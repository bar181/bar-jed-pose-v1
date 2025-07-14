# Key Pose Detection Repositories - Quick Reference

## Browser-Based Solutions (WebAssembly/WebGL/WebGPU)

### 1. **MediaPipe** ⭐ Recommended for production
- **Repository**: https://github.com/google-ai-edge/mediapipe
- **Features**: 33 3D keypoints, WebAssembly backend, real-time performance
- **Browser Setup**: 
  ```javascript
  npm install @mediapipe/pose
  ```
- **Live Demo**: https://mediapipe-studio.webapps.google.com/demo/pose_landmarker

### 2. **TensorFlow.js Pose Detection** ⭐ Best for flexibility
- **Repository**: https://github.com/tensorflow/tfjs-models/tree/master/pose-detection
- **Models**: MoveNet (fastest), BlazePose (most keypoints), PoseNet (multi-person)
- **Browser Setup**:
  ```javascript
  npm install @tensorflow-models/pose-detection
  npm install @tensorflow/tfjs-core @tensorflow/tfjs-converter
  npm install @tensorflow/tfjs-backend-webgl
  ```
- **Live Demo**: https://storage.googleapis.com/tfjs-models/demos/pose-detection/index.html

### 3. **MediaPipe JavaScript Examples**
- **Repository**: https://github.com/LintangWisesa/MediaPipe-in-JavaScript
- **Features**: Ready-to-use HTML examples
- **Best for**: Quick prototyping

### 4. **WASM Object Detection Example**
- **Repository**: https://github.com/martishin/wasm-object-detection
- **Tech Stack**: Rust + WebAssembly + React
- **Best for**: Understanding WASM integration

## Advanced Research Repositories

### 5. **OpenPose** (Server-side, most accurate)
- **Repository**: https://github.com/CMU-Perceptual-Computing-Lab/openpose
- **Features**: 135 keypoints (body + face + hands)
- **Note**: C++, requires GPU, not browser-native

### 6. **MMPose** (Comprehensive toolkit)
- **Repository**: https://github.com/open-mmlab/mmpose
- **Features**: 2D/3D pose, animal pose, extensive model zoo
- **Note**: Python-based, research-oriented

### 7. **Detectron2** (Facebook Research)
- **Repository**: https://github.com/facebookresearch/detectron2
- **Features**: DensePose for surface detection
- **Note**: PyTorch-based, state-of-the-art accuracy

## Specialized Implementations

### 8. **BodyPose3D** (3D pose from 2D)
- **Repository**: https://github.com/TemugeB/bodypose3d
- **Features**: Real-time 3D pose estimation using MediaPipe
- **Tech**: Python + MediaPipe

### 9. **DepthaAI BlazePose**
- **Repository**: https://github.com/geaxgx/depthai_blazepose
- **Features**: Hardware-accelerated pose detection
- **Hardware**: Works with OAK cameras

### 10. **PoseNet WebGL Demos**
- **Repository**: https://github.com/jscriptcoder/tfjs-posenet
- **Features**: Minimal WebGL implementation
- **Best for**: Learning WebGL integration

## Quick Start Code Snippets

### MediaPipe (WebAssembly)
```javascript
const pose = new Pose({
  locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`
});

await pose.initialize();

const results = await pose.process(imageElement);
if (results.poseLandmarks) {
  console.log('Detected pose:', results.poseLandmarks);
}
```

### TensorFlow.js MoveNet (WebGL/WebGPU)
```javascript
const model = poseDetection.SupportedModels.MoveNet;
const detector = await poseDetection.createDetector(model, {
  modelType: poseDetection.movenet.modelType.SINGLEPOSE_LIGHTNING,
  enableSmoothing: true
});

const poses = await detector.estimatePoses(video);
```

### Performance Comparison Table

| Solution | FPS | Keypoints | Backend | Best For |
|----------|-----|-----------|---------|----------|
| MoveNet Lightning | 50+ | 17 | WebGL | Real-time apps |
| MoveNet Thunder | 30+ | 17 | WebGL | Accuracy + speed |
| BlazePose Lite | 40+ | 33 | WASM | Full body tracking |
| BlazePose Full | 25+ | 33 | WASM | Detailed pose |
| PoseNet MobileNet | 20+ | 17 | WebGL | Multi-person |

## Additional Resources

- **TensorFlow.js WebGPU Backend**: https://github.com/tensorflow/tfjs/tree/master/tfjs-backend-webgpu
- **WebAssembly SIMD**: https://v8.dev/features/simd
- **MediaPipe Model Cards**: https://developers.google.com/mediapipe/solutions/vision/pose_landmarker
- **Pose Estimation Papers**: https://paperswithcode.com/task/pose-estimation

## Community & Support

- **MediaPipe Discord**: https://discord.gg/mediapipe
- **TensorFlow.js Discussions**: https://github.com/tensorflow/tfjs/discussions
- **Stack Overflow Tags**: `mediapipe`, `tensorflowjs`, `pose-detection`