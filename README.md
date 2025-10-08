# Reload Task Trigger — Qlik Sense Extension

**TaskTrigger** is a lightweight Qlik Sense visualization extension that:
- Starts a **primary QRS task** with one click.
- Optionally shows the **latest status** for the primary task and a **second task** (if configured).

---

## ✨ Features

- ▶️ **Start reload** for a given Task ID (QMC → Tasks → Copy ID).
- 📊 **Check status** (Primary + optional Second task): *Status, Started, Completed, Duration*.
- Uses **QRS API** on the same origin as the Hub/QMC.
- Automatically handles **XSRF protection (xrfkey)** in both query and header.
- Optional **Virtual Proxy prefix** (e.g., `/jwt`).

---

## 🧩 Requirements

- **Qlik Sense Enterprise on Windows (client-managed)**.
- The extension is used **inside the Qlik Sense Hub** so QRS is same-origin (credentials included).
- Your user or virtual proxy must have permission to call:
  - `POST /qrs/task/{id}/start`
  - `GET  /qrs/reloadtask/{id}`

---

## 📦 Installation

1. Create a folder named `TaskTrigger`.
2. Add the files:

```
TaskTrigger/
├─ TaskTrigger.js
├─ TaskTrigger.qext
└─ preview.png   (optional)
```

3. Example `TaskTrigger.qext`:

```json
{
  "name": "Reload Task Trigger",
  "version": "1.0.0",
  "type": "visualization",
  "description": "Start and monitor up to two QMC reload tasks from a Qlik Sense sheet",
  "author": "Your Name",
  "icon": "extension",
  "preview": "preview.png"
}
```

4. Upload via **QMC → Extensions**

5. Open a sheet and add the visualization **Reload Task Trigger**.

---

## ⚙️ Properties

| Property | Description | Default |
|-----------|--------------|----------|
| **Primary Task ID (GUID)** | Main task to start and monitor | — |
| **Second Task ID (GUID)** | Optional, only monitored | — |
| **Virtual Proxy Prefix** | e.g., `/jwt` or blank | `""` |
| **Start Button Label** | Custom label for start button | `▶️ Start Reload Task` |
| **Check Button Label** | Label for check button | `📊 Check status (both)` |

> The second task is **not started directly** — it’s only monitored (e.g., if triggered by the first one).

---

## 🔌 How It Works

The helper `qrsFetch(vpx, path, opts)`:
- Generates a random `xrfkey` (16 chars).
- Adds it to both **query string** and **HTTP header**.
- Calls QRS API endpoints using `fetch()` with `credentials: "include"`.

---

### Endpoints Used
- **Start task**: `POST /qrs/task/{taskId}/start?xrfkey={key}`
- **Read last result**: `GET /qrs/reloadtask/{taskId}?xrfkey={key}`
- The task status retrieval is performed via the endpoint  
  `/qrs/reloadtask/{id}` (newer Qlik builds).  
  If you are using an older Qlik Sense build, change this path to  
  `/qrs/task/{id}` (older builds).

---

### Security Notes
- Uses `fetch` with `credentials: "include"`; runs inside the Hub so it inherits auth cookies.
- Generates and sends `xrfkey` with each QRS request.

---

### Troubleshooting
- **HTTP 403 / 401**: Verify your user/proxy has rights on QRS endpoints.
- **HTTP 404**: Check the Task ID (must be a valid GUID from QMC).
- **“Invalid Task ID”** in UI: Copy *exactly* from QMC → Tasks → Copy ID.
- **No “Completed/Duration” while running**: This is by design; these appear only when the task is not running.

---

## ⚙️ **TaskTrigger.js**

```javascript
// TaskTrigger.js
define(["qlik"], function (qlik) {
  "use strict";

  // ---------- Helpers ----------
  function fmtDate(s){
    if (!s) return "—";
    const d = new Date(s);
    if (isNaN(d.getTime())) return String(s) || "—";
    return d.toLocaleString("en-GB", {
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
      hour12: false
    });
  }

  function fmtMs(ms){
    if (ms == null) return "—";
    const sec = Math.max(0, Math.round(ms/1000));
    const h = Math.floor(sec/3600), m = Math.floor((sec%3600)/60), s = sec%60;
    return `${h}h ${m}m ${s}s`;
  }

  const statusMap = {
    0:"Unknown", 1:"Triggered", 2:"Started/Running", 3:"Queued",
    6:"Aborted", 7:"Succeeded", 8:"Failed", 9:"Skipped", 10:"Retrying"
  };

  function makeXrf(){
    const a="abcdefghijklmnopqrstuvwxyz0123456789";
    let x=""; for(let i=0;i<16;i++) x+=a[Math.floor(Math.random()*a.length)];
    return x;
  }

  async function qrsFetch(vpx, path, opts){
    const key = makeXrf();
    const sep = path.indexOf("?") >= 0 ? "&" : "?";
    const url = `${(vpx||"")}${path}${sep}xrfkey=${key}`;
    const headers = Object.assign({
      "X-Qlik-Xrfkey": key,
      "Accept": "application/json"
    }, (opts && opts.headers) || {});
    const init = Object.assign({
      credentials: "include",
      cache: "no-store",
      headers
    }, opts || {});
    return fetch(url, init);
  }

  async function fetchTaskLastResult(vpx, taskId){
    const res = await qrsFetch(vpx, `/qrs/reloadtask/${taskId}`, { method:"GET" });
    const txt = await res.text();

    if (!res.ok) {
      return { name: taskId, error: `HTTP ${res.status} ${res.statusText}\n${txt || ""}` };
    }

    let t=null;
    try { t = txt ? JSON.parse(txt) : null; } catch {}

    const name = t?.name || taskId;
    const ler  = t?.operational?.lastExecutionResult || t?.operational?.lastExecution || null;
    if (!ler) return { name, empty:true };

    const start = ler.executionStartTime || ler.startTime || null;
    const stop  = ler.executionStopTime  || ler.stopTime  || null;
    const stNum = (typeof ler.status === "number") ? ler.status : null;
    const stTxt = ler.statusText || (stNum!=null ? (statusMap[stNum] || `Status #${stNum}`) : "-");

    const running = (stNum === 2) || (!!start && !stop) || /running/i.test(stTxt || "");

    let duration = "—";
    try {
      if (start && stop) duration = fmtMs(new Date(stop) - new Date(start));
      else if (running && start) duration = fmtMs(Date.now() - new Date(start));
    } catch {}

    return { name, statusText: stTxt, start, stop, duration, running };
  }

  // ---------- Property panel ----------
  const props = {
    type: "items",
    component: "accordion",
    items: {
      settings: { uses: "settings" },
      main: {
        label: "TaskTrigger",
        type: "items",
        items: {
          taskId: {
            ref: "props.taskId",
            label: "Primary Task ID",
            type: "string",
            expression: "optional"
          },
          secondTaskId: {
            ref: "props.secondTaskId",
            label: "Second Task ID – optional",
            type: "string",
            expression: "optional"
          },
          vpx: {
            ref: "props.vpx",
            label: "Virtual proxy prefix (e.g., /jwt or empty)",
            type: "string",
            defaultValue: ""
          },
          startLabel: {
            ref: "props.startLabel",
            label: "Start button label",
            type: "string",
            defaultValue: "▶️ Start Reload"
          },
          checkLabel: {
            ref: "props.checkLabel",
            label: "Check status (both) button label",
            type: "string",
            defaultValue: "📊 Check Status"
          }
        }
      }
    }
  };

  // ---------- Paint ----------
  function paint($element, layout) {
    const el = $element[0];
    const id = "dtt_" + layout.qInfo.qId;

    const taskId1 = (layout.props && layout.props.taskId) || "";
    const taskId2 = (layout.props && layout.props.secondTaskId) || "";
    const vpx     = (layout.props && layout.props.vpx) || "";

    const startLabel = (layout.props && layout.props.startLabel) || "▶️ Start Reload";
    const checkLabel = (layout.props && layout.props.checkLabel) || "📊 Check Status";

    el.innerHTML = `
      <div style="font:14px/1.4 system-ui,Segoe UI,Arial;min-height:110px">
        <div style="display:flex;gap:8px;flex-wrap:wrap;margin-bottom:8px">
          <button id="${id}_start" style="padding:8px 12px;border:0;border-radius:8px;background:#0a7c2f;color:#fff;cursor:pointer">${startLabel}</button>
          <button id="${id}_check" style="padding:8px 12px;border:1px solid #ccc;border-radius:8px;background:#fff;cursor:pointer">${checkLabel}</button>
        </div>
        <pre id="${id}_log" style="margin:0 0 6px 0;white-space:pre-wrap"></pre>
        <pre id="${id}_status" style="margin:0;white-space:pre-wrap"></pre>
      </div>
    `;

    const $log = el.querySelector(`#${id}_log`);
    const $status = el.querySelector(`#${id}_status`);
    const setLog = (s)=> { $log.textContent = s || ""; };
    const setStatus = (s)=> { $status.textContent = s || ""; };

    if (!taskId1) {
      setLog("⚠️ Please set the Primary Task ID in the object properties.");
      return qlik.Promise.resolve();
    }

    el.querySelector(`#${id}_start`).onclick = async () => {
      setLog("Starting task 1…");
      try {
        const res = await qrsFetch(vpx, `/qrs/task/${taskId1}/start`, { method: "POST" });
        const txt = await res.text();
        let ok = res.ok;
        try {
          const j = txt ? JSON.parse(txt) : null;
          if (j && typeof j.value === "string" && j.value.toLowerCase() === taskId1.toLowerCase()) ok = true;
        } catch {}
        if (ok) setLog(`✅ Success: Task 1 has started.`);
        else    setLog(`⚠️ Failed to start Task 1.\nHTTP ${res.status} ${res.statusText}\n${txt}`);
      } catch (e) {
        setLog("❌ Network error: " + e.message);
      }
    };

    el.querySelector(`#${id}_check`).onclick = async () => {
      const ids = [taskId1].concat(taskId2 ? [taskId2] : []);
      setStatus("Fetching status…");

      try {
        const parts = [];
        for (const tId of ids) {
          if (!/^[0-9a-fA-F-]{36}$/.test(tId)) {
            parts.push(`• ${tId}\n  ⚠️ Invalid Task ID. Copy it from QMC → Tasks → Copy ID.`);
            continue;
          }

          const r = await fetchTaskLastResult(vpx, tId);

          if (r.error) {
            parts.push(`• ${r.name}\n  ⚠️ Read error:\n  ${r.error}`);
          } else if (r.empty) {
            parts.push(`• ${r.name}\n  ℹ️ No previous execution found.`);
          } else {
            const lines = [
              `• ${r.name}`,
              `  Status: ${r.statusText}`,
              `  Started: ${fmtDate(r.start)}`
            ];
            if (!r.running) {
              lines.push(`  Completed: ${fmtDate(r.stop)}`);
              lines.push(`  Duration: ${r.duration}`);
            }
            parts.push(lines.join("\n"));
          }
        }
        setStatus(`📊 Tasks status:\n${parts.join("\n\n")}`);
      } catch (e) {
        setStatus("❌ Network error: " + e.message);
      }
    };

    return qlik.Promise.resolve();
  }

  return {
    definition: props,
    paint,
    support: { snapshot: false, export: false, exportData: false }
  };
});
```

---

### License

GNU General Public License v3.0 (GPL-3.0)

---

### Credits
Built for Qlik admins who need one-click triggering and quick status checks directly from a sheet.
