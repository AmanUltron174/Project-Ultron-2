"use client";

import { useEffect, useRef, useState } from "react";
import { OrbScene } from "@/lib/orbScene";
import { HandTracker } from "@/lib/handTracker";

export default function JarvisOrb() {
  const containerRef = useRef<HTMLDivElement>(null);
  const videoRef = useRef<HTMLVideoElement>(null);
  const sceneRef = useRef<OrbScene | null>(null);
  const trackerRef = useRef<HandTracker | null>(null);

  const [gesturesEnabled, setGesturesEnabled] = useState(false);
  const [statusText, setStatusText] = useState("SYSTEM READY");

  useEffect(() => {
    if (!containerRef.current) return;

    // Initialize Three.js Scene
    const orbScene = new OrbScene(containerRef.current);
    sceneRef.current = orbScene;

    // Mouse Controls
    let isDragging = false;
    let previousMouse = { x: 0, y: 0 };

    const handleMouseDown = (e: MouseEvent) => {
      isDragging = true;
      previousMouse = { x: e.clientX, y: e.clientY };
    };

    const handleMouseMove = (e: MouseEvent) => {
      if (!isDragging || !sceneRef.current) return;
      const deltaX = e.clientX - previousMouse.x;
      const deltaY = e.clientY - previousMouse.y;

      sceneRef.current.rotate(deltaX * 0.005, deltaY * 0.005);
      previousMouse = { x: e.clientX, y: e.clientY };
    };

    const handleMouseUp = () => (isDragging = false);

    const handleWheel = (e: WheelEvent) => {
      if (!sceneRef.current) return;
      sceneRef.current.zoom(e.deltaY * 0.001);
    };

    const domElem = containerRef.current;
    domElem.addEventListener("mousedown", handleMouseDown);
    window.addEventListener("mousemove", handleMouseMove);
    window.addEventListener("mouseup", handleMouseUp);
    domElem.addEventListener("wheel", handleWheel);

    // Keyboard Shortcuts
    const handleKeyDown = (e: KeyboardEvent) => {
      const key = e.key.toLowerCase();
      if (key === "g") toggleGestures();
      if (key === "r" && sceneRef.current) sceneRef.current.resetView();
      if (key === "+" || key === "=") sceneRef.current?.zoom(-0.2);
      if (key === "-") sceneRef.current?.zoom(0.2);
    };

    window.addEventListener("keydown", handleKeyDown);

    return () => {
      orbScene.dispose();
      domElem.removeEventListener("mousedown", handleMouseDown);
      window.removeEventListener("mousemove", handleMouseMove);
      window.removeEventListener("mouseup", handleMouseUp);
      domElem.removeEventListener("wheel", handleWheel);
      window.removeEventListener("keydown", handleKeyDown);
    };
  }, []);

  const toggleGestures = async () => {
    setGesturesEnabled((prev) => {
      const nextState = !prev;
      if (nextState) {
        startTracking();
      } else {
        stopTracking();
      }
      return nextState;
    });
  };

  const startTracking = async () => {
    if (!videoRef.current || !sceneRef.current) return;
    setStatusText("INITIALIZING CAMERA & MEDIAPIPE...");

    try {
      const tracker = new HandTracker(videoRef.current, (gestureData) => {
        if (!sceneRef.current) return;

        if (gestureData.type === "PAN" && gestureData.delta) {
          sceneRef.current.rotate(gestureData.delta.x * 2, gestureData.delta.y * 2);
        } else if (gestureData.type === "ZOOM" && gestureData.zoomFactor !== undefined) {
          sceneRef.current.zoom(gestureData.zoomFactor * 0.05);
        }
      });

      await tracker.start();
      trackerRef.current = tracker;
      setStatusText("GESTURE CONTROL ACTIVE");
    } catch (err) {
      console.error(err);
      setStatusText("CAMERA ACCESS DENIED / FAILED");
      setGesturesEnabled(false);
    }
  };

  const stopTracking = () => {
    trackerRef.current?.stop();
    trackerRef.current = null;
    setStatusText("SYSTEM READY");
  };

  return (
    <div style={{ position: "relative", width: "100vw", height: "100vh", background: "#050811", overflow: "hidden" }}>
      {/* Three.js Viewport */}
      <div ref={containerRef} style={{ width: "100%", height: "100%", cursor: "grab" }} />

      {/* Hidden Video element for MediaPipe tracking */}
      <video ref={videoRef} style={{ display: "none" }} playsInline />

      {/* Sci-Fi HUD Overlay */}
      <div style={{ position: "absolute", top: 20, left: 20, color: "#00f0ff", fontFamily: "monospace", zIndex: 10 }}>
        <h1 style={{ margin: 0, textShadow: "0 0 10px #00f0ff" }}>ULTRON ORB UI</h1>
        <p style={{ margin: "5px 0", fontSize: "0.8rem", opacity: 0.8 }}>{statusText}</p>
      </div>

      <div style={{ position: "absolute", bottom: 20, left: 20, zIndex: 10 }}>
        <button
          onClick={toggleGestures}
          style={{
            background: gesturesEnabled ? "#00f0ff" : "transparent",
            color: gesturesEnabled ? "#000" : "#00f0ff",
            border: "1px solid #00f0ff",
            padding: "10px 20px",
            fontFamily: "monospace",
            fontWeight: "bold",
            cursor: "pointer",
            boxShadow: gesturesEnabled ? "0 0 15px #00f0ff" : "none",
            transition: "all 0.3s ease"
          }}
        >
          {gesturesEnabled ? "GESTURES ON [G]" : "GESTURES OFF [G]"}
        </button>
      </div>
    </div>
  );
}


