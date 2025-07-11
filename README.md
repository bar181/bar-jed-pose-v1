# Pose Detection

A real-time webcam-based human pose detection and motion tracking system built with React, TypeScript, and TensorFlow.js.

## 🎯 Features

- **Real-time Pose Detection**: Uses TensorFlow.js with MoveNet for accurate pose estimation
- **Motion Tracking**: Comprehensive movement analysis including joint positions, velocity, and acceleration
- **Gait Analysis**: Specialized algorithms for walking pattern detection and analysis
- **Visual Feedback**: Live skeleton overlay and motion visualization
- **Performance Optimized**: Runs at 60+ FPS on modern hardware
- **Docker Support**: Containerized deployment with runtime path configuration
- **PWA Ready**: Installable as a Progressive Web App

## 🚀 Quick Start

### Prerequisites

- Node.js 18+ and npm
- Webcam access
- Modern browser with WebGL support

### Installation

```bash
# Clone the repository
git clone https://github.com/jedarden/pose-detection.git
cd pose-detection

# Install dependencies
npm install

# Start development server
npm run dev
```

Visit http://localhost:5173 to see the application.

### Docker Deployment

```bash
# Build the container
docker build -t pose-detection .

# Run with default settings (root path)
docker run -p 8080:80 pose-detection

# Run with custom base path
docker run -p 8080:80 -e BASE_PATH=/pose pose-detection
```

## 🏗️ Architecture

- **Frontend**: React 18 with TypeScript
- **Pose Detection**: TensorFlow.js with MoveNet
- **State Management**: React hooks and context
- **Styling**: CSS modules with responsive design
- **Build Tool**: Vite for fast development
- **Testing**: Vitest for unit tests, Cypress for E2E

## 📦 Key Components

- **PoseDetector**: Core pose detection engine
- **MotionTracker**: Movement analysis and tracking
- **GaitAnalyzer**: Walking pattern detection
- **VisualOverlay**: Real-time skeleton rendering
- **MetricsDisplay**: Performance and accuracy metrics

## 🔧 Configuration

### Environment Variables

- `BASE_PATH`: URL base path for deployment (default: `/`)
- `VITE_API_URL`: Backend API endpoint (optional)

### Runtime Configuration

The application supports runtime path configuration for flexible deployment:

```javascript
// Automatically detects and configures base path
const basePath = window.location.pathname.match(/^\/[^\/]+/)?.[0] || '/';
```

## 📊 Performance

- Targets 60 FPS for smooth motion tracking
- Optimized pose detection pipeline
- WebGL acceleration for TensorFlow.js
- Efficient canvas rendering

## 🧪 Testing

```bash
# Run unit tests
npm test

# Run with coverage
npm run test:coverage

# Run E2E tests
npm run test:e2e
```

## 🚢 Deployment

### Kubernetes

See `k8s-example.yaml` for a complete deployment example with:
- Deployment with health checks
- Service configuration
- Ingress with path-based routing
- TLS certificate management

### Docker Compose

```yaml
version: '3.8'
services:
  pose-detection:
    image: pose-detection:latest
    ports:
      - "8080:80"
    environment:
      - BASE_PATH=/pose
```

## 📄 License

MIT License - see LICENSE file for details

## 🤝 Contributing

Contributions are welcome! Please read our contributing guidelines and submit pull requests to our repository.

## 📞 Support

- Create an issue on GitHub
- Check the documentation in `/docs`
- View deployment guide in `DEPLOYMENT.md`

---

Built with ❤️ using React, TypeScript, and TensorFlow.js