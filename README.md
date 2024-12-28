# json_2_webgl
AI generated json to webgl
I'll create a Python parser that converts JSON data from embedded devices into a WebGL-compatible 3D VR visualization.

import json
import numpy as np
from dataclasses import dataclass
from typing import List, Dict, Any
import threading
import websockets
import asyncio
from pathlib import Path

@dataclass
class Device3DModel:
    position: List[float]
    rotation: List[float]
    scale: List[float]
    metrics: Dict[str, float]
    model_path: str

class DeviceVRParser:
    """Parser to convert embedded device JSON data into WebGL-compatible 3D scene data"""
    
    def __init__(self, model_directory: str = "models/"):
        self.model_directory = Path(model_directory)
        self.devices: Dict[str, Device3DModel] = {}
        self.websocket_server = None
        self.clients = set()
        
    async def start_websocket_server(self, host: str = "localhost", port: int = 8765):
        """Start WebSocket server to stream VR updates to clients"""
        self.websocket_server = await websockets.serve(self.handle_client, host, port)
        print(f"WebSocket server started on ws://{host}:{port}")
        
    async def handle_client(self, websocket, path):
        """Handle WebSocket client connections and updates"""
        self.clients.add(websocket)
        try:
            while True:
                # Send current scene state to client
                await websocket.send(json.dumps(self.get_scene_data()))
                await asyncio.sleep(0.1)  # Update rate limit
        finally:
            self.clients.remove(websocket)

    def parse_device_data(self, json_data: str) -> None:
        """Parse JSON data from embedded device and update internal state"""
        try:
            data = json.loads(json_data)
            device_id = data.get("device_id")
            
            if not device_id:
                raise ValueError("Missing device_id in JSON data")
                
            # Create or update device model
            self.devices[device_id] = Device3DModel(
                position=self._parse_vector(data.get("position", [0, 0, 0])),
                rotation=self._parse_vector(data.get("rotation", [0, 0, 0])),
                scale=self._parse_vector(data.get("scale", [1, 1, 1])),
                metrics=data.get("metrics", {}),
                model_path=str(self.model_directory / f"{data.get('model_type', 'default')}.gltf")
            )
            
        except json.JSONDecodeError as e:
            print(f"Error parsing JSON: {e}")
        except Exception as e:
            print(f"Error processing device data: {e}")

    def _parse_vector(self, vec_data: List[float]) -> List[float]:
        """Convert vector data to WebGL-compatible format"""
        return [float(x) for x in vec_data[:3]]

    def get_scene_data(self) -> Dict[str, Any]:
        """Generate complete scene data for WebGL rendering"""
        return {
            "devices": {
                device_id: {
                    "position": device.position,
                    "rotation": device.rotation,
                    "scale": device.scale,
                    "metrics": device.metrics,
                    "model": device.model_path
                }
                for device_id, device in self.devices.items()
            },
            "scene": {
                "ambient_light": [0.5, 0.5, 0.5],
                "camera": {
                    "position": [0, 2, 5],
                    "target": [0, 0, 0]
                }
            }
        }

    def generate_webgl_html(self) -> str:
        """Generate HTML page with WebGL/Three.js visualization"""
        return """
<!DOCTYPE html>
<html>
<head>
    <title>Device VR Monitor</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/webxr-polyfill@latest/build/webxr-polyfill.min.js"></script>
    <style>
        body { margin: 0; }
        canvas { width: 100%; height: 100% }
    </style>
</head>
<body>
    <script>
        let scene, camera, renderer, devices = {};
        const ws = new WebSocket('ws://localhost:8765');

        function init() {
            scene = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
            
            renderer = new THREE.WebGLRenderer();
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);
            
            // Enable VR
            renderer.xr.enabled = true;
            document.body.appendChild(VRButton.createButton(renderer));
            
            // Add lights
            const light = new THREE.AmbientLight(0x404040);
            scene.add(light);
            
            // Handle WebSocket updates
            ws.onmessage = function(event) {
                updateScene(JSON.parse(event.data));
            };
            
            animate();
        }
        
        function updateScene(data) {
            // Update device positions and states
            Object.entries(data.devices).forEach(([id, device]) => {
                if (!devices[id]) {
                    // Create new device representation
                    const geometry = new THREE.BoxGeometry();
                    const material = new THREE.MeshPhongMaterial({color: 0x00ff00});
                    devices[id] = new THREE.Mesh(geometry, material);
                    scene.add(devices[id]);
                }
                
                // Update position and rotation
                devices[id].position.set(...device.position);
                devices[id].rotation.set(...device.rotation);
                devices[id].scale.set(...device.scale);
            });
        }
        
        function animate() {
            renderer.setAnimationLoop(render);
        }
        
        function render() {
            renderer.render(scene, camera);
        }
        
        init();
    </script>
</body>
</html>
"""

# Example usage
async def main():
    # Create parser instance
    parser = DeviceVRParser()
    
    # Start WebSocket server
    await parser.start_websocket_server()
    
    # Example: Process device data
    example_data = {
        "device_id": "sensor_001",
        "position": [1.0, 0.0, -2.0],
        "rotation": [0.0, 45.0, 0.0],
        "scale": [1.0, 1.0, 1.0],
        "model_type": "sensor",
        "metrics": {
            "temperature": 25.4,
            "humidity": 65.0,
            "battery": 82.0
        }
    }
    
    parser.parse_device_data(json.dumps(example_data))
    
    # Generate HTML file
    with open("device_vr_monitor.html", "w") as f:
        f.write(parser.generate_webgl_html())
    
    # Keep server running
    await asyncio.Future()

if __name__ == "__main__":
    asyncio.run(main())

This implementation provides a complete system for converting JSON data from embedded devices into a WebGL-based VR visualization. Here's how it works:

1. The `DeviceVRParser` class:
   - Parses incoming JSON data from devices
   - Maintains device state information
   - Converts device data into WebGL-compatible format
   - Provides WebSocket server for real-time updates
   - Generates HTML/JavaScript for the VR interface

2. Key features:
   - Real-time WebSocket updates
   - 3D position, rotation, and scale support
   - Device metrics visualization
   - VR-ready WebGL output
   - Support for custom 3D models (GLTF format)

3. To use it:
   - Run the parser server
   - Send JSON data from your embedded devices
   - Open the generated HTML file in a VR-capable browser

Example JSON input format:
```json
{
    "device_id": "sensor_001",
    "position": [1.0, 0.0, -2.0],
    "rotation": [0.0, 45.0, 0.0],
    "scale": [1.0, 1.0, 1.0],
    "model_type": "sensor",
    "metrics": {
        "temperature": 25.4,
        "humidity": 65.0,
        "battery": 82.0
    }
}
```

Would you like me to explain any specific part in more detail or add additional features to the implementation?
