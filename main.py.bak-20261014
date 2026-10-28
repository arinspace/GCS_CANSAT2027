#!/usr/bin/env python3
"""
CanSat 2027 Ground Control Station — Team 1059 — web server

    python main.py                      # nothing streams until the operator runs the CMD sequence
    python main.py --virtual            # loopback virtual CanSat ready; still nothing streams
    python main.py --virtual --auto-arm # scripted demo: SIM -> SIM ENABLE -> SIM ACTIVATE
    python main.py --config my.json --profile judges.csv --log

Open http://127.0.0.1:8000 . The ground-station logic lives in gcs_core.py
(GroundStation); this file only serves the page and relays JSON over /ws.
"""
import argparse
import asyncio
import json
import os
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import FileResponse, PlainTextResponse, Response
from fastapi.staticfiles import StaticFiles

import gcs_core as core

HERE = os.path.dirname(os.path.abspath(__file__))
INDEX_HTML = os.path.join(HERE, "templates", "index.html")
STATIC_DIR = os.path.join(HERE, "static")
DEMO_PROFILE = os.path.join(HERE, "profiles", "sim_profile_1059.csv")

CLIENT_QUEUE_MAX = 5000      # per-browser outbound backlog; a browser that falls this far behind is dropped (it reconnects and gets a fresh snapshot)


def parse_args(argv=None):
    ap = argparse.ArgumentParser(
        description=f"CanSat 2027 Ground Control Station — Team {core.TEAM_ID} (web)")
    ap.add_argument("--virtual", "--sim", dest="virtual", action="store_true",
                    help="start the loopback virtual CanSat (no hardware needed). "
                         "Nothing streams until the operator presses SIM -> SIM ENABLE "
                         "-> SIM ACTIVATE")
    ap.add_argument("--auto-arm", action="store_true",
                    help="with --virtual: load the bundled demo profile and run "
                         "SIM -> SIM ENABLE -> SIM ACTIVATE by script (demo only)")
    ap.add_argument("--manual-arm", action="store_true",
                    help="no effect (manual arming is now the default); kept so old "
                         "command lines still work")
    ap.add_argument("--sim-speed", type=float, default=1.0,
                    help="simulator speed multiplier (2.0 = twice as fast)")
    ap.add_argument("--config", default=core.CONFIG_FILE,
                    help="radio config file (default: radio_config.json next to gcs_core.py)")
    ap.add_argument("--profile", default=None,
                    help="load a pressure profile file at startup")
    ap.add_argument("--log", action="store_true",
                    help="start CSV logging at startup")
    ap.add_argument("--host", default="127.0.0.1")
    ap.add_argument("--port", type=int, default=8000)
    return ap.parse_args(argv)


class Hub:
    """Fan-out of JSON messages to every connected browser."""

    def __init__(self):
        self.clients = set()

    def broadcast(self, msg):
        try:
            text = json.dumps(msg, allow_nan=False)
        except (TypeError, ValueError) as e:
            print(f"[hub] dropped unserialisable {msg.get('type')} message: {e}")
            return
        for c in list(self.clients):
            c.push(text)


class Client:
    def __init__(self, ws, hub):
        self.ws = ws
        self.hub = hub
        self.q = asyncio.Queue(maxsize=CLIENT_QUEUE_MAX)

    def push(self, text):
        try:
            self.q.put_nowait(text)
        except asyncio.QueueFull:
            # too slow to keep up — disconnect; the page reconnects and resyncs
            self.hub.clients.discard(self)
            asyncio.get_running_loop().create_task(self.ws.close(code=1013))

    async def sender(self):
        try:
            while True:
                await self.ws.send_text(await self.q.get())
        except Exception:
            self.hub.clients.discard(self)      # socket gone; receive loop ends too


OPTS = parse_args([])        # defaults when imported (e.g. `uvicorn main:app`)
hub = Hub()


@asynccontextmanager
async def lifespan(app):
    gs = core.GroundStation(config_path=OPTS.config, sim_speed=OPTS.sim_speed)
    gs.attach(asyncio.get_running_loop(), hub.broadcast)
    autoarm = OPTS.virtual and OPTS.auto_arm
    profile = OPTS.profile
    if profile is None and autoarm and os.path.exists(DEMO_PROFILE):
        profile = DEMO_PROFILE
    gs.start(autostart_sim=OPTS.virtual, profile=profile, log_csv=OPTS.log,
             autoarm=autoarm)
    app.state.gs = gs
    try:
        yield
    finally:
        gs.shutdown()


app = FastAPI(title=f"CanSat 2027 GCS — Team {core.TEAM_ID}", lifespan=lifespan)
app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")


@app.get("/")
async def index():
    # no-cache: a stale cached page after an update would talk an old protocol
    return FileResponse(INDEX_HTML, media_type="text/html",
                        headers={"Cache-Control": "no-cache"})


@app.get("/api/log/{vehicle}.csv")
async def history_log(vehicle: str):
    """'Save Log Container' / 'Save Log PQ' — the display source's current history."""
    v = vehicle.upper()
    if v not in ("C", "P"):
        return PlainTextResponse("vehicle must be C or P", status_code=404)
    out = app.state.gs.history_csv(v)
    if out is None:
        return PlainTextResponse("No data in history yet", status_code=404)
    name, text = out
    return Response(text, media_type="text/csv",
                    headers={"Content-Disposition": f'attachment; filename="{name}"'})


@app.get("/tiles/{z}/{x}/{y}")
async def map_tile(z: int, x: int, y: int):
    """MAP: OFFLINE — one tile from map_cache.db (tkintermapview schema)."""
    data = await asyncio.to_thread(core.read_cached_tile, z, x, y)
    if data is None:
        return Response(status_code=404)
    mt = "image/png" if data[:8] == b"\x89PNG\r\n\x1a\n" else "image/jpeg"
    return Response(data, media_type=mt)


@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    await ws.accept()
    gs = app.state.gs
    client = Client(ws, hub)
    # no await between registering and queueing the snapshot: nothing can be
    # broadcast in between, so the browser sees an unbroken stream
    hub.clients.add(client)
    client.push(json.dumps(gs.config(), allow_nan=False))
    client.push(json.dumps(gs.snapshot(), allow_nan=False))
    sender = asyncio.create_task(client.sender())
    try:
        while True:
            text = await ws.receive_text()
            try:
                msg = json.loads(text)
            except ValueError:
                continue
            if isinstance(msg, dict):
                try:
                    gs.handle_message(msg)
                except Exception as e:          # never let one bad message drop the socket
                    gs._log(f"browser message error ({msg.get('type')}): {e}", "err")
    except (WebSocketDisconnect, RuntimeError):
        pass
    finally:
        hub.clients.discard(client)
        sender.cancel()


def main():
    global OPTS
    OPTS = parse_args()
    uvicorn.run(app, host=OPTS.host, port=OPTS.port, log_level="info")


if __name__ == "__main__":
    main()
