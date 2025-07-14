# Integration Test Plans - Phase 3 Testing

## 🔗 Comprehensive Integration Testing Strategy

### Integration Testing Framework
```typescript
// integration.config.ts - Specialized integration test configuration
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/integration-setup.ts'],
    globals: true,
    testTimeout: 30000, // Extended timeout for integration tests
    threads: false, // Run integration tests sequentially
    coverage: {
      provider: 'v8',
      include: [
        'src/services/**/*.ts',
        'src/components/**/*.tsx',
        'src/hooks/**/*.ts'
      ],
      exclude: ['src/**/*.test.ts', 'src/**/*.spec.ts']
    }
  }
});
```

## 🎯 Service Integration Tests

### Camera-Pose Detection Pipeline Integration
```typescript
// CameraPoseIntegration.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { CameraService } from '../services/CameraService';
import { PoseDetectionService } from '../services/PoseDetectionService';
import { PoseSmoothingService } from '../services/PoseSmoothingService';
import { PoseValidationService } from '../services/PoseValidationService';

describe('Camera-Pose Detection Pipeline Integration', () => {
  let cameraService: CameraService;
  let poseService: PoseDetectionService;
  let smoothingService: PoseSmoothingService;
  let validationService: PoseValidationService;
  let mockStream: MediaStream;

  beforeEach(async () => {
    // Setup services
    cameraService = new CameraService();
    poseService = new PoseDetectionService();
    smoothingService = new PoseSmoothingService();
    validationService = new PoseValidationService();

    // Mock camera stream
    mockStream = createMockMediaStream();
    vi.spyOn(navigator.mediaDevices, 'getUserMedia').mockResolvedValue(mockStream);

    // Initialize pose detection
    await poseService.initialize({
      modelType: 'lightning',
      enableGPU: false, // Use CPU for consistent testing
      inputResolution: { width: 192, height: 192 },
      maxPoses: 1,
      validation: { minPoseConfidence: 0.3, minKeypointConfidence: 0.3 },
      smoothing: { enabled: true, smoothingFactor: 0.5, minConfidence: 0.3, historySize: 5 },
      performance: { targetFPS: 30, enableFrameSkipping: false, frameSkipInterval: 1 }
    });
  });

  afterEach(() => {
    cameraService.cleanup();
    poseService.dispose();
    smoothingService.cleanup();
  });

  it('should complete end-to-end pose detection workflow', async () => {
    // Step 1: Initialize camera
    const stream = await cameraService.getStream({
      video: { width: 640, height: 480 }
    });
    expect(stream).toBeDefined();

    // Step 2: Create video element from stream
    const videoElement = createVideoElementFromStream(stream);
    await waitForVideoReady(videoElement);

    // Step 3: Detect poses
    const rawPoses = await poseService.detectPoses(videoElement);
    expect(rawPoses).toBeInstanceOf(Array);

    // Step 4: Validate poses
    const validatedPoses = validationService.validatePoses(rawPoses);
    expect(validatedPoses.length).toBeLessThanOrEqual(rawPoses.length);

    // Step 5: Apply smoothing
    const smoothedPoses = smoothingService.smoothPoses(validatedPoses);
    expect(smoothedPoses).toBeInstanceOf(Array);

    // Verify end-to-end data integrity
    if (smoothedPoses.length > 0) {
      expect(smoothedPoses[0]).toMatchObject({
        keypoints: expect.any(Array),
        confidence: expect.any(Number),
        timestamp: expect.any(Number)
      });
    }
  });

  it('should handle camera permission failures gracefully', async () => {
    vi.spyOn(navigator.mediaDevices, 'getUserMedia')
      .mockRejectedValue(new DOMException('Permission denied', 'NotAllowedError'));

    await expect(cameraService.getStream({
      video: { width: 640, height: 480 }
    })).rejects.toThrow('Permission denied');

    // System should remain stable after permission failure
    expect(poseService.isReady()).toBe(true);
  });

  it('should maintain synchronization between camera and pose detection', async () => {
    const stream = await cameraService.getStream({
      video: { width: 640, height: 480 }
    });
    
    const videoElement = createVideoElementFromStream(stream);
    await waitForVideoReady(videoElement);

    const detectionPromises: Promise<any>[] = [];
    const timestamps: number[] = [];

    // Run multiple detections to check synchronization
    for (let i = 0; i < 5; i++) {
      const timestamp = Date.now();
      timestamps.push(timestamp);
      detectionPromises.push(poseService.detectPoses(videoElement));
      
      // Small delay between detections
      await new Promise(resolve => setTimeout(resolve, 50));
    }

    const results = await Promise.all(detectionPromises);

    // Verify all detections completed successfully
    expect(results).toHaveLength(5);
    
    // Verify temporal consistency
    for (let i = 1; i < results.length; i++) {
      if (results[i].length > 0 && results[i-1].length > 0) {
        const currentPose = results[i][0];
        const previousPose = results[i-1][0];
        
        // Timestamps should be monotonically increasing
        expect(currentPose.timestamp).toBeGreaterThan(previousPose.timestamp);
      }
    }
  });

  it('should handle camera stream interruption and recovery', async () => {
    const stream = await cameraService.getStream({
      video: { width: 640, height: 480 }
    });
    
    const videoElement = createVideoElementFromStream(stream);
    await waitForVideoReady(videoElement);

    // Initial detection should work
    const initialPoses = await poseService.detectPoses(videoElement);
    expect(initialPoses).toBeInstanceOf(Array);

    // Simulate stream interruption
    stream.getTracks().forEach(track => track.stop());

    // Detection should handle gracefully
    const posesAfterInterruption = await poseService.detectPoses(videoElement);
    expect(posesAfterInterruption).toBeInstanceOf(Array);

    // Recovery: Get new stream
    const newStream = await cameraService.getStream({
      video: { width: 640, height: 480 }
    });
    
    const newVideoElement = createVideoElementFromStream(newStream);
    await waitForVideoReady(newVideoElement);

    // Detection should work with new stream
    const recoveredPoses = await poseService.detectPoses(newVideoElement);
    expect(recoveredPoses).toBeInstanceOf(Array);
  });

  it('should validate performance under load', async () => {
    const stream = await cameraService.getStream({
      video: { width: 640, height: 480 }
    });
    
    const videoElement = createVideoElementFromStream(stream);
    await waitForVideoReady(videoElement);

    const iterations = 30; // 1 second worth of frames at 30 FPS
    const startTime = performance.now();
    const processingTimes: number[] = [];

    for (let i = 0; i < iterations; i++) {
      const frameStart = performance.now();
      await poseService.detectPoses(videoElement);
      const frameEnd = performance.now();
      
      processingTimes.push(frameEnd - frameStart);
    }

    const totalTime = performance.now() - startTime;
    const avgProcessingTime = processingTimes.reduce((a, b) => a + b, 0) / processingTimes.length;
    const maxProcessingTime = Math.max(...processingTimes);

    // Performance assertions
    expect(avgProcessingTime).toBeLessThan(33); // 30 FPS budget
    expect(maxProcessingTime).toBeLessThan(50); // Allow some variance
    expect(totalTime / iterations).toBeLessThan(40); // Overall throughput
  });
});

// Test utility functions
function createMockMediaStream(): MediaStream {
  const stream = new MediaStream();
  const track = new MediaStreamTrack();
  
  // Mock track properties
  Object.defineProperty(track, 'kind', { value: 'video' });
  Object.defineProperty(track, 'readyState', { value: 'live' });
  Object.defineProperty(track, 'enabled', { value: true });
  
  stream.addTrack(track);
  return stream;
}

function createVideoElementFromStream(stream: MediaStream): HTMLVideoElement {
  const video = document.createElement('video');
  video.srcObject = stream;
  video.width = 640;
  video.height = 480;
  video.autoplay = true;
  video.muted = true;
  
  return video;
}

async function waitForVideoReady(video: HTMLVideoElement): Promise<void> {
  return new Promise((resolve) => {
    video.addEventListener('loadedmetadata', () => {
      resolve();
    });
    
    // Fallback timeout
    setTimeout(resolve, 1000);
  });
}
```

### Gait Analysis Integration Tests
```typescript
// GaitAnalysisIntegration.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { PoseDetectionService } from '../services/PoseDetectionService';
import { GaitAnalysisService } from '../services/GaitAnalysisService';
import { DataExportService } from '../services/DataExportService';

describe('Gait Analysis Integration', () => {
  let poseService: PoseDetectionService;
  let gaitService: GaitAnalysisService;
  let exportService: DataExportService;

  beforeEach(async () => {
    poseService = new PoseDetectionService();
    gaitService = new GaitAnalysisService();
    exportService = new DataExportService();

    await poseService.initialize({
      modelType: 'lightning',
      enableGPU: false,
      inputResolution: { width: 192, height: 192 },
      maxPoses: 1,
      validation: { minPoseConfidence: 0.3, minKeypointConfidence: 0.3 },
      smoothing: { enabled: true, smoothingFactor: 0.5, minConfidence: 0.3, historySize: 5 },
      performance: { targetFPS: 30, enableFrameSkipping: false, frameSkipInterval: 1 }
    });

    // Setup gait calibration
    gaitService.calibrate({
      pixelsPerMeter: 100,
      referenceHeight: 1.7,
      cameraHeight: 1.0,
      cameraAngle: 0
    });
  });

  afterEach(() => {
    poseService.dispose();
    gaitService.reset();
  });

  it('should integrate pose detection with gait analysis', async () => {
    const mockVideoElement = createMockVideoElement();

    // Simulate walking motion over time
    const walkingMotion = generateWalkingMotionData();
    
    for (const motionFrame of walkingMotion) {
      // Mock pose detection to return specific pose data
      vi.spyOn(poseService, 'detectPoses').mockResolvedValueOnce([{
        keypoints: motionFrame.keypoints,
        confidence: 0.8,
        timestamp: motionFrame.timestamp,
        id: 'test-pose',
        boundingBox: { x: 0, y: 0, width: 100, height: 200 }
      }]);

      const poses = await poseService.detectPoses(mockVideoElement);
      
      // Feed poses to gait analysis
      if (poses.length > 0) {
        const tfPose = convertToTensorFlowPose(poses[0]);
        gaitService.addPose(tfPose, motionFrame.timestamp);
      }
    }

    // Analyze gait parameters
    const gaitParams = gaitService.calculateGaitParameters();

    // Verify gait analysis results
    expect(gaitParams.confidence).toBeGreaterThan(0);
    expect(gaitParams.cadence).toBeGreaterThan(0);
    expect(gaitParams.strideLength).toBeGreaterThan(0);
  });

  it('should handle real-time gait event detection', async () => {
    const mockVideoElement = createMockVideoElement();
    const eventHistory: any[] = [];

    // Simulate real-time detection loop
    const walkingSequence = generateRealtimeWalkingSequence();
    
    for (const frame of walkingSequence) {
      vi.setSystemTime(frame.timestamp);
      
      vi.spyOn(poseService, 'detectPoses').mockResolvedValueOnce([{
        keypoints: frame.keypoints,
        confidence: 0.8,
        timestamp: frame.timestamp,
        id: 'test-pose',
        boundingBox: { x: 0, y: 0, width: 100, height: 200 }
      }]);

      const poses = await poseService.detectPoses(mockVideoElement);
      
      if (poses.length > 0) {
        const tfPose = convertToTensorFlowPose(poses[0]);
        gaitService.addPose(tfPose, frame.timestamp);
        
        // Check for new gait events
        const recentEvents = gaitService.getRecentEvents(1000);
        eventHistory.push(...recentEvents);
      }
    }

    // Verify event detection
    const heelStrikes = eventHistory.filter(e => e.type === 'heel-strike');
    const toeOffs = eventHistory.filter(e => e.type === 'toe-off');

    expect(heelStrikes.length).toBeGreaterThan(0);
    expect(toeOffs.length).toBeGreaterThan(0);
  });

  it('should export comprehensive gait analysis data', async () => {
    // Setup analysis data
    await simulateGaitAnalysisSession();

    const gaitParams = gaitService.calculateGaitParameters();
    const gaitEvents = gaitService.getRecentEvents();
    const poseHistory = gaitService.getPoseHistory();

    // Export data
    const exportData = await exportService.exportGaitAnalysis({
      parameters: gaitParams,
      events: gaitEvents,
      poseHistory: poseHistory,
      metadata: {
        sessionId: 'test-session',
        duration: 30000,
        calibration: gaitService.getCalibration()
      }
    });

    // Verify export structure
    expect(exportData).toMatchObject({
      format: 'gait-analysis',
      version: expect.any(String),
      timestamp: expect.any(Number),
      data: {
        parameters: expect.objectContaining({
          cadence: expect.any(Number),
          strideLength: expect.any(Number),
          velocity: expect.any(Number)
        }),
        events: expect.any(Array),
        poseHistory: expect.any(Array),
        metadata: expect.any(Object)
      }
    });

    // Verify data integrity
    expect(exportData.data.events.length).toBeGreaterThan(0);
    expect(exportData.data.poseHistory.length).toBeGreaterThan(0);
  });

  it('should maintain temporal consistency across services', async () => {
    const mockVideoElement = createMockVideoElement();
    const timestamps: number[] = [];
    const poses: any[] = [];

    // Generate sequential poses with precise timing
    for (let i = 0; i < 10; i++) {
      const timestamp = Date.now() + (i * 100); // 100ms intervals
      timestamps.push(timestamp);

      vi.setSystemTime(timestamp);
      
      const mockPose = createMockPoseData(timestamp);
      vi.spyOn(poseService, 'detectPoses').mockResolvedValueOnce([mockPose]);

      const detectedPoses = await poseService.detectPoses(mockVideoElement);
      poses.push(...detectedPoses);

      // Add to gait analysis
      if (detectedPoses.length > 0) {
        const tfPose = convertToTensorFlowPose(detectedPoses[0]);
        gaitService.addPose(tfPose, timestamp);
      }
    }

    // Verify temporal consistency
    for (let i = 1; i < poses.length; i++) {
      expect(poses[i].timestamp).toBeGreaterThan(poses[i-1].timestamp);
    }

    // Verify gait service maintains temporal order
    const poseHistory = gaitService.getPoseHistory();
    for (let i = 1; i < poseHistory.length; i++) {
      expect(poseHistory[i].timestamp).toBeGreaterThan(poseHistory[i-1].timestamp);
    }
  });

  // Helper functions
  async function simulateGaitAnalysisSession(): Promise<void> {
    const duration = 10000; // 10 seconds
    const fps = 30;
    const frames = (duration / 1000) * fps;

    for (let i = 0; i < frames; i++) {
      const timestamp = Date.now() + (i * (1000 / fps));
      const walkingPose = generateWalkingPoseData(timestamp, i);
      
      gaitService.addPose(walkingPose, timestamp);
    }
  }

  function generateWalkingMotionData() {
    const frames = [];
    const duration = 5000; // 5 seconds
    const fps = 30;
    const frameCount = (duration / 1000) * fps;

    for (let i = 0; i < frameCount; i++) {
      const timestamp = Date.now() + (i * (1000 / fps));
      const phase = (i / frameCount) * Math.PI * 4; // 2 complete stride cycles
      
      frames.push({
        timestamp,
        keypoints: generateWalkingKeypoints(phase)
      });
    }

    return frames;
  }

  function generateRealtimeWalkingSequence() {
    // Generate realistic walking sequence with heel strikes and toe offs
    const sequence = [];
    const baseTime = Date.now();
    
    // Left heel strike cycle
    sequence.push({
      timestamp: baseTime,
      keypoints: generateKeypointsForPhase('left-heel-strike')
    });
    
    sequence.push({
      timestamp: baseTime + 200,
      keypoints: generateKeypointsForPhase('left-stance')
    });
    
    sequence.push({
      timestamp: baseTime + 400,
      keypoints: generateKeypointsForPhase('left-toe-off')
    });
    
    // Right heel strike cycle
    sequence.push({
      timestamp: baseTime + 600,
      keypoints: generateKeypointsForPhase('right-heel-strike')
    });
    
    sequence.push({
      timestamp: baseTime + 800,
      keypoints: generateKeypointsForPhase('right-stance')
    });
    
    sequence.push({
      timestamp: baseTime + 1000,
      keypoints: generateKeypointsForPhase('right-toe-off')
    });

    return sequence;
  }
});

// Mock data generators
function generateWalkingKeypoints(phase: number) {
  return [
    { x: 320, y: 50, score: 0.9, name: 'nose' },
    { x: 310, y: 40, score: 0.8, name: 'left_eye' },
    { x: 330, y: 40, score: 0.8, name: 'right_eye' },
    { x: 300, y: 60, score: 0.7, name: 'left_ear' },
    { x: 340, y: 60, score: 0.7, name: 'right_ear' },
    { x: 280, y: 120, score: 0.9, name: 'left_shoulder' },
    { x: 360, y: 120, score: 0.9, name: 'right_shoulder' },
    { x: 260, y: 180, score: 0.8, name: 'left_elbow' },
    { x: 380, y: 180, score: 0.8, name: 'right_elbow' },
    { x: 240, y: 220, score: 0.7, name: 'left_wrist' },
    { x: 400, y: 220, score: 0.7, name: 'right_wrist' },
    { x: 300, y: 200, score: 0.9, name: 'left_hip' },
    { x: 340, y: 200, score: 0.9, name: 'right_hip' },
    { x: 290, y: 320, score: 0.8, name: 'left_knee' },
    { x: 350, y: 320, score: 0.8, name: 'right_knee' },
    { x: 280 + Math.sin(phase) * 20, y: 440, score: 0.9, name: 'left_ankle' },
    { x: 360 + Math.sin(phase + Math.PI) * 20, y: 440, score: 0.9, name: 'right_ankle' }
  ];
}

function generateKeypointsForPhase(phase: string) {
  const baseKeypoints = generateWalkingKeypoints(0);
  
  if (phase === 'left-heel-strike') {
    baseKeypoints[15].y = 450; // Left ankle lower
  } else if (phase === 'left-toe-off') {
    baseKeypoints[15].y = 430; // Left ankle higher
  } else if (phase === 'right-heel-strike') {
    baseKeypoints[16].y = 450; // Right ankle lower
  } else if (phase === 'right-toe-off') {
    baseKeypoints[16].y = 430; // Right ankle higher
  }
  
  return baseKeypoints;
}

function convertToTensorFlowPose(poseResult: any) {
  return {
    keypoints: poseResult.keypoints,
    score: poseResult.confidence
  };
}

function createMockPoseData(timestamp: number) {
  return {
    keypoints: generateWalkingKeypoints(0),
    confidence: 0.8,
    timestamp,
    id: `pose-${timestamp}`,
    boundingBox: { x: 0, y: 0, width: 100, height: 200 }
  };
}

function generateWalkingPoseData(timestamp: number, frameIndex: number) {
  const phase = (frameIndex / 30) * Math.PI; // Assume 30 FPS
  return {
    keypoints: generateWalkingKeypoints(phase),
    score: 0.8
  };
}

function createMockVideoElement(): HTMLVideoElement {
  const video = document.createElement('video');
  video.width = 640;
  video.height = 480;
  video.videoWidth = 640;
  video.videoHeight = 480;
  return video;
}
```

## 🎨 Component Integration Tests

### PoseOverlay-CameraView Integration
```typescript
// PoseOverlayCameraIntegration.test.tsx
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import { CameraView } from '../components/Camera/CameraView';
import { PoseOverlay } from '../components/PoseOverlay';
import { ApplicationProvider } from '../context/ApplicationContext';

describe('PoseOverlay-CameraView Integration', () => {
  beforeEach(() => {
    // Mock MediaDevices API
    Object.defineProperty(navigator, 'mediaDevices', {
      value: {
        getUserMedia: vi.fn().mockResolvedValue(createMockMediaStream()),
        enumerateDevices: vi.fn().mockResolvedValue([])
      },
      configurable: true
    });

    // Mock canvas context
    const mockContext = {
      clearRect: vi.fn(),
      drawImage: vi.fn(),
      stroke: vi.fn(),
      beginPath: vi.fn(),
      moveTo: vi.fn(),
      lineTo: vi.fn(),
      arc: vi.fn(),
      fill: vi.fn()
    };

    HTMLCanvasElement.prototype.getContext = vi.fn().mockReturnValue(mockContext);
  });

  it('should render camera view with pose overlay', async () => {
    const mockPoses = [createMockPoseData()];
    const mockVideoElement = createMockVideoElement();

    render(
      <ApplicationProvider>
        <div style={{ position: 'relative' }}>
          <CameraView
            resolution={{ width: 640, height: 480 }}
            onStreamReady={vi.fn()}
            onError={vi.fn()}
          />
          <PoseOverlay
            poses={mockPoses}
            videoElement={mockVideoElement}
            settings={{
              showKeypoints: true,
              showSkeleton: true,
              confidenceThreshold: 0.3,
              skeletonColor: '#00ff00',
              keypointColor: '#ff0000'
            }}
          />
        </div>
      </ApplicationProvider>
    );

    await waitFor(() => {
      expect(screen.getByRole('video')).toBeInTheDocument();
      expect(screen.getByRole('img')).toBeInTheDocument(); // Canvas with img role
    });
  });

  it('should synchronize pose overlay with video dimensions', async () => {
    const mockPoses = [createMockPoseData()];
    const { rerender } = render(
      <ApplicationProvider>
        <div style={{ position: 'relative' }}>
          <CameraView
            resolution={{ width: 640, height: 480 }}
            onStreamReady={vi.fn()}
            onError={vi.fn()}
          />
          <PoseOverlay
            poses={mockPoses}
            videoElement={createMockVideoElement()}
            settings={{
              showKeypoints: true,
              showSkeleton: true,
              confidenceThreshold: 0.3,
              skeletonColor: '#00ff00',
              keypointColor: '#ff0000'
            }}
          />
        </div>
      </ApplicationProvider>
    );

    // Change video resolution
    rerender(
      <ApplicationProvider>
        <div style={{ position: 'relative' }}>
          <CameraView
            resolution={{ width: 1280, height: 720 }}
            onStreamReady={vi.fn()}
            onError={vi.fn()}
          />
          <PoseOverlay
            poses={mockPoses}
            videoElement={createMockVideoElement(1280, 720)}
            settings={{
              showKeypoints: true,
              showSkeleton: true,
              confidenceThreshold: 0.3,
              skeletonColor: '#00ff00',
              keypointColor: '#ff0000'
            }}
          />
        </div>
      </ApplicationProvider>
    );

    await waitFor(() => {
      const canvas = screen.getByRole('img') as HTMLCanvasElement;
      expect(canvas.width).toBe(1280);
      expect(canvas.height).toBe(720);
    });
  });
});
```

This comprehensive integration testing plan ensures all components work together seamlessly with proper error handling and performance validation.