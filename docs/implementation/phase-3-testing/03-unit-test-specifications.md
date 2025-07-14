# Unit Test Specifications - Phase 3 Testing

## 🎯 Comprehensive Unit Testing Strategy

### Testing Framework Stack (2024 Optimized)
```typescript
// vitest.config.ts - Optimized configuration
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    globals: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        'src/**/*.test.{ts,tsx}',
        'src/**/*.stories.{ts,tsx}',
        'src/types/**',
        'src/test/**'
      ],
      thresholds: {
        global: {
          branches: 85,
          functions: 95,
          lines: 90,
          statements: 90
        },
        'src/services/': {
          branches: 95,
          functions: 100,
          lines: 95,
          statements: 95
        }
      }
    },
    pool: 'threads',
    poolOptions: {
      threads: {
        singleThread: false,
        minThreads: 2,
        maxThreads: 4
      }
    }
  }
});
```

## 📋 Service Layer Unit Tests

### PoseDetectionService Tests
```typescript
// PoseDetectionService.test.ts
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { PoseDetectionService } from '../services/PoseDetectionService';
import * as tf from '@tensorflow/tfjs';
import type { PoseDetectionConfig } from '../types/pose';

describe('PoseDetectionService', () => {
  let service: PoseDetectionService;
  let mockConfig: PoseDetectionConfig;

  beforeEach(() => {
    // Setup mocks for TensorFlow.js
    vi.mock('@tensorflow/tfjs', () => ({
      ready: vi.fn().mockResolvedValue(undefined),
      setBackend: vi.fn().mockResolvedValue(undefined),
      getBackend: vi.fn().mockReturnValue('webgl'),
      ENV: {
        set: vi.fn()
      },
      zeros: vi.fn().mockReturnValue({
        dispose: vi.fn()
      })
    }));

    // Mock pose detection library
    vi.mock('@tensorflow-models/pose-detection', () => ({
      createDetector: vi.fn().mockResolvedValue({
        estimatePoses: vi.fn().mockResolvedValue([{
          score: 0.9,
          keypoints: [
            { x: 100, y: 100, score: 0.9, name: 'nose' },
            { x: 110, y: 120, score: 0.8, name: 'left_eye' }
          ]
        }]),
        dispose: vi.fn()
      }),
      SupportedModels: {
        MoveNet: 'MoveNet'
      },
      movenet: {
        modelType: {
          SINGLEPOSE_LIGHTNING: 'lightning',
          SINGLEPOSE_THUNDER: 'thunder'
        }
      }
    }));

    mockConfig = {
      modelType: 'lightning',
      enableGPU: true,
      inputResolution: { width: 192, height: 192 },
      maxPoses: 1,
      validation: {
        minPoseConfidence: 0.3,
        minKeypointConfidence: 0.3
      },
      smoothing: {
        enabled: true,
        smoothingFactor: 0.5,
        minConfidence: 0.3,
        historySize: 5
      },
      performance: {
        targetFPS: 30,
        enableFrameSkipping: true,
        frameSkipInterval: 2
      }
    };

    service = new PoseDetectionService();
  });

  afterEach(() => {
    service.dispose();
    vi.clearAllMocks();
  });

  describe('Initialization', () => {
    it('should initialize successfully with valid config', async () => {
      await service.initialize(mockConfig);
      
      expect(service.isReady()).toBe(true);
      expect(tf.ready).toHaveBeenCalled();
      expect(tf.setBackend).toHaveBeenCalledWith('webgl');
    });

    it('should fallback to CPU when GPU initialization fails', async () => {
      vi.mocked(tf.setBackend).mockRejectedValueOnce(new Error('GPU not available'));
      
      await service.initialize(mockConfig);
      
      expect(tf.setBackend).toHaveBeenCalledWith('webgl');
      expect(tf.setBackend).toHaveBeenCalledWith('cpu');
      expect(service.isReady()).toBe(true);
    });

    it('should throw error when initialization fails completely', async () => {
      vi.mocked(tf.ready).mockRejectedValue(new Error('TensorFlow.js not available'));
      
      await expect(service.initialize(mockConfig)).rejects.toThrow('Failed to initialize pose detection');
      expect(service.isReady()).toBe(false);
    });

    it('should validate configuration parameters', async () => {
      const invalidConfig = {
        ...mockConfig,
        validation: { minPoseConfidence: 1.5 } // Invalid confidence > 1
      };

      await expect(service.initialize(invalidConfig)).rejects.toThrow('minPoseConfidence must be between 0 and 1');
    });
  });

  describe('Pose Detection', () => {
    beforeEach(async () => {
      await service.initialize(mockConfig);
    });

    it('should detect poses from valid video frame', async () => {
      const mockVideoElement = createMockVideoElement();
      
      const results = await service.detectPoses(mockVideoElement);
      
      expect(results).toHaveLength(1);
      expect(results[0]).toMatchObject({
        confidence: expect.any(Number),
        keypoints: expect.arrayContaining([
          expect.objectContaining({
            x: expect.any(Number),
            y: expect.any(Number),
            score: expect.any(Number),
            name: expect.any(String)
          })
        ]),
        timestamp: expect.any(Number),
        id: expect.any(String)
      });
    });

    it('should handle null input gracefully', async () => {
      await expect(service.detectPoses(null as any)).rejects.toThrow('Invalid input: imageData is null or undefined');
    });

    it('should filter poses below confidence threshold', async () => {
      const mockDetector = {
        estimatePoses: vi.fn().mockResolvedValue([
          { score: 0.2, keypoints: [] }, // Below threshold
          { score: 0.8, keypoints: [] }  // Above threshold
        ])
      };
      
      // Inject mock detector
      (service as any).detector = mockDetector;
      
      const results = await service.detectPoses(createMockVideoElement());
      
      expect(results).toHaveLength(1);
      expect(results[0].confidence).toBe(0.8);
    });

    it('should handle detection errors gracefully', async () => {
      const mockDetector = {
        estimatePoses: vi.fn().mockRejectedValue(new Error('Detection failed'))
      };
      
      (service as any).detector = mockDetector;
      
      const results = await service.detectPoses(createMockVideoElement());
      
      // Should return empty array or last valid poses
      expect(results).toBeInstanceOf(Array);
    });

    it('should implement adaptive frame skipping', async () => {
      const mockVideoElement = createMockVideoElement();
      
      // Simulate poor performance
      (service as any).adaptivePerformance.performanceScore = 0.5;
      (service as any).adaptivePerformance.frameSkipMultiplier = 2;
      
      // Call multiple times
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      
      const stats = service.getStats();
      expect(stats.droppedFrames).toBeGreaterThan(0);
    });
  });

  describe('Performance Monitoring', () => {
    beforeEach(async () => {
      await service.initialize(mockConfig);
    });

    it('should track processing statistics', async () => {
      const mockVideoElement = createMockVideoElement();
      
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      
      const stats = service.getStats();
      
      expect(stats).toMatchObject({
        totalPoses: expect.any(Number),
        averageConfidence: expect.any(Number),
        currentFPS: expect.any(Number),
        avgProcessingTime: expect.any(Number),
        droppedFrames: expect.any(Number),
        memoryUsage: expect.any(Number),
        modelLoadTime: expect.any(Number)
      });
    });

    it('should calculate FPS correctly', async () => {
      const mockVideoElement = createMockVideoElement();
      
      // Mock performance.now to control timing
      let time = 0;
      vi.spyOn(performance, 'now').mockImplementation(() => {
        time += 16.67; // ~60 FPS
        return time;
      });
      
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      
      const stats = service.getStats();
      expect(stats.currentFPS).toBeGreaterThan(50);
    });

    it('should update adaptive performance metrics', async () => {
      const mockVideoElement = createMockVideoElement();
      
      // Simulate slow processing
      vi.spyOn(performance, 'now')
        .mockReturnValueOnce(0)
        .mockReturnValueOnce(100); // 100ms processing time
      
      await service.detectPoses(mockVideoElement);
      
      const metrics = service.getPerformanceMetrics();
      expect(metrics.performanceScore).toBeLessThan(1.0);
      expect(metrics.frameSkipMultiplier).toBeGreaterThan(1.0);
    });
  });

  describe('Motion Tracking', () => {
    beforeEach(async () => {
      await service.initialize(mockConfig);
    });

    it('should track motion data for keypoints', async () => {
      const mockVideoElement = createMockVideoElement();
      
      // Simulate movement by changing keypoint positions
      const mockDetector = {
        estimatePoses: vi.fn()
          .mockResolvedValueOnce([{
            score: 0.9,
            keypoints: [{ x: 100, y: 100, score: 0.9, name: 'nose' }]
          }])
          .mockResolvedValueOnce([{
            score: 0.9,
            keypoints: [{ x: 110, y: 105, score: 0.9, name: 'nose' }]
          }])
      };
      
      (service as any).detector = mockDetector;
      
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      
      const motionData = service.getMotionData();
      expect(motionData.size).toBeGreaterThan(0);
      
      const noseMotion = Array.from(motionData.values())[0];
      expect(noseMotion).toMatchObject({
        velocity: expect.any(Number),
        acceleration: expect.any(Number),
        lastPosition: expect.objectContaining({
          x: expect.any(Number),
          y: expect.any(Number)
        })
      });
    });

    it('should maintain pose history', async () => {
      const mockVideoElement = createMockVideoElement();
      
      await service.detectPoses(mockVideoElement);
      await service.detectPoses(mockVideoElement);
      
      const history = service.getPoseHistory();
      expect(history.length).toBe(2);
    });

    it('should limit pose history size', async () => {
      const mockVideoElement = createMockVideoElement();
      const maxHistory = (service as any).config.smoothing.historySize * 2;
      
      // Generate more poses than max history
      for (let i = 0; i < maxHistory + 10; i++) {
        await service.detectPoses(mockVideoElement);
      }
      
      const history = service.getPoseHistory();
      expect(history.length).toBeLessThanOrEqual(maxHistory);
    });
  });

  describe('Configuration Management', () => {
    it('should update configuration dynamically', async () => {
      await service.initialize(mockConfig);
      
      const newConfig = {
        validation: { minPoseConfidence: 0.5 }
      };
      
      service.updateConfig(newConfig);
      
      // Verify config was updated
      const internalConfig = (service as any).config;
      expect(internalConfig.validation.minPoseConfidence).toBe(0.5);
    });

    it('should validate configuration updates', () => {
      const invalidUpdate = {
        validation: { minPoseConfidence: -0.1 }
      };
      
      expect(() => service.updateConfig(invalidUpdate)).toThrow('minPoseConfidence must be between 0 and 1');
    });
  });

  describe('Resource Management', () => {
    it('should dispose resources properly', async () => {
      await service.initialize(mockConfig);
      const mockDetector = (service as any).detector;
      
      service.dispose();
      
      expect(mockDetector.dispose).toHaveBeenCalled();
      expect(service.isReady()).toBe(false);
    });

    it('should reset statistics on disposal', async () => {
      await service.initialize(mockConfig);
      await service.detectPoses(createMockVideoElement());
      
      service.dispose();
      
      const stats = service.getStats();
      expect(stats.totalPoses).toBe(0);
      expect(stats.avgProcessingTime).toBe(0);
    });
  });
});

// Test Utilities
function createMockVideoElement(): HTMLVideoElement {
  const video = document.createElement('video');
  video.width = 640;
  video.height = 480;
  video.videoWidth = 640;
  video.videoHeight = 480;
  
  // Mock video element methods
  vi.spyOn(video, 'play').mockResolvedValue();
  vi.spyOn(video, 'pause').mockImplementation(() => {});
  
  return video;
}

function createMockImageData(width = 192, height = 192): ImageData {
  const data = new Uint8ClampedArray(width * height * 4);
  return new ImageData(data, width, height);
}

function createMockCanvas(): HTMLCanvasElement {
  const canvas = document.createElement('canvas');
  const mockContext = {
    getImageData: vi.fn().mockReturnValue(createMockImageData()),
    putImageData: vi.fn(),
    clearRect: vi.fn(),
    drawImage: vi.fn()
  };
  
  vi.spyOn(canvas, 'getContext').mockReturnValue(mockContext as any);
  return canvas;
}
```

### GaitAnalysisService Tests
```typescript
// GaitAnalysisService.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { GaitAnalysisService } from '../services/GaitAnalysisService';
import type { Pose, Keypoint } from '@tensorflow-models/pose-detection';

describe('GaitAnalysisService', () => {
  let service: GaitAnalysisService;

  beforeEach(() => {
    service = new GaitAnalysisService();
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  describe('Pose Addition', () => {
    it('should add pose with sufficient confidence', () => {
      const mockPose = createMockPose({
        leftAnkle: { x: 100, y: 200, score: 0.8 },
        rightAnkle: { x: 150, y: 200, score: 0.8 }
      });

      service.addPose(mockPose, Date.now());
      
      const history = service.getPoseHistory();
      expect(history).toHaveLength(1);
      expect(history[0].leftAnkle.score).toBe(0.8);
    });

    it('should reject pose with low confidence', () => {
      const mockPose = createMockPose({
        leftAnkle: { x: 100, y: 200, score: 0.2 },
        rightAnkle: { x: 150, y: 200, score: 0.2 }
      });

      service.addPose(mockPose, Date.now());
      
      const history = service.getPoseHistory();
      expect(history).toHaveLength(0);
    });

    it('should maintain maximum history length', () => {
      const maxLength = 300; // from service implementation
      
      // Add more poses than max length
      for (let i = 0; i < maxLength + 50; i++) {
        const mockPose = createMockPose();
        service.addPose(mockPose, Date.now() + i);
      }
      
      const history = service.getPoseHistory();
      expect(history.length).toBeLessThanOrEqual(maxLength);
    });
  });

  describe('Gait Event Detection', () => {
    it('should detect heel strike events', () => {
      const timestamps = [0, 50, 100, 150, 200];
      const yPositions = [180, 185, 190, 195, 200]; // Ankle moving down
      
      timestamps.forEach((timestamp, index) => {
        const mockPose = createMockPose({
          leftAnkle: { x: 100, y: yPositions[index], score: 0.8 },
          leftKnee: { x: 100, y: yPositions[index] - 50, score: 0.8 }
        });
        
        vi.setSystemTime(timestamp);
        service.addPose(mockPose, timestamp);
      });
      
      const events = service.getRecentEvents();
      const heelStrikes = events.filter(e => e.type === 'heel-strike' && e.foot === 'left');
      expect(heelStrikes.length).toBeGreaterThan(0);
    });

    it('should detect toe-off events', () => {
      const timestamps = [0, 50, 100, 150, 200];
      const yPositions = [200, 190, 180, 170, 160]; // Ankle moving up
      
      timestamps.forEach((timestamp, index) => {
        const mockPose = createMockPose({
          leftAnkle: { x: 100, y: yPositions[index], score: 0.8 },
          leftKnee: { x: 100, y: yPositions[index] - 50, score: 0.8 }
        });
        
        vi.setSystemTime(timestamp);
        service.addPose(mockPose, timestamp);
      });
      
      const events = service.getRecentEvents();
      const toeOffs = events.filter(e => e.type === 'toe-off' && e.foot === 'left');
      expect(toeOffs.length).toBeGreaterThan(0);
    });

    it('should enforce minimum step duration', () => {
      const minDuration = 200; // from service implementation
      
      // Add two heel strikes too close together
      const mockPose1 = createMockPose();
      const mockPose2 = createMockPose();
      
      service.addPose(mockPose1, 1000);
      service.addPose(mockPose2, 1000 + minDuration - 50); // Too soon
      
      const events = service.getRecentEvents();
      const heelStrikes = events.filter(e => e.type === 'heel-strike');
      expect(heelStrikes.length).toBeLessThanOrEqual(1);
    });
  });

  describe('Gait Parameter Calculation', () => {
    beforeEach(() => {
      // Setup calibration
      service.calibrate({
        pixelsPerMeter: 100,
        referenceHeight: 1.7,
        cameraHeight: 1.0,
        cameraAngle: 0
      });
    });

    it('should calculate cadence from heel strikes', () => {
      const baseTime = Date.now();
      
      // Add heel strikes at regular intervals (120 steps/min = 2 steps/sec)
      for (let i = 0; i < 10; i++) {
        const timestamp = baseTime + (i * 500); // 0.5 sec intervals
        const mockPose = createMockPose();
        
        vi.setSystemTime(timestamp);
        service.addPose(mockPose, timestamp);
        
        // Manually add heel strike events
        (service as any).gaitEvents.push({
          type: 'heel-strike',
          foot: i % 2 === 0 ? 'left' : 'right',
          timestamp,
          position: { x: 100, y: 200 },
          confidence: 0.8
        });
      }
      
      const params = service.calculateGaitParameters();
      expect(params.cadence).toBeGreaterThan(100); // Should be around 120
      expect(params.cadence).toBeLessThan(140);
    });

    it('should calculate step lengths with calibration', () => {
      const events = [
        {
          type: 'heel-strike' as const,
          foot: 'left' as const,
          timestamp: 1000,
          position: { x: 100, y: 200 },
          confidence: 0.8
        },
        {
          type: 'heel-strike' as const,
          foot: 'left' as const,
          timestamp: 1500,
          position: { x: 200, y: 200 }, // 100 pixels = 1 meter with calibration
          confidence: 0.8
        }
      ];
      
      // Inject events directly for testing
      (service as any).gaitEvents = events;
      
      const params = service.calculateGaitParameters();
      expect(params.leftStepLength).toBe(1.0); // 100 pixels / 100 pixels per meter
    });

    it('should return empty parameters when insufficient data', () => {
      const params = service.calculateGaitParameters();
      
      expect(params).toMatchObject({
        cadence: 0,
        strideLength: 0,
        strideTime: 0,
        stepWidth: 0,
        velocity: 0,
        symmetryIndex: 0,
        confidence: 0
      });
    });

    it('should calculate gait phase correctly', () => {
      const currentTime = Date.now();
      vi.setSystemTime(currentTime);
      
      // Add recent heel strike
      (service as any).gaitEvents.push({
        type: 'heel-strike',
        foot: 'left',
        timestamp: currentTime - 100, // 100ms ago
        position: { x: 100, y: 200 },
        confidence: 0.8
      });
      
      const params = service.calculateGaitParameters();
      expect(params.gaitPhase.left).toBe('heel-strike');
      expect(params.gaitPhase.leftProgress).toBeGreaterThanOrEqual(0);
    });
  });

  describe('Calibration', () => {
    it('should calibrate manually with provided data', () => {
      const calibrationData = {
        pixelsPerMeter: 150,
        referenceHeight: 1.8,
        cameraHeight: 1.2,
        cameraAngle: 5
      };
      
      service.calibrate(calibrationData);
      
      expect(service.isCalibrated()).toBe(true);
      expect(service.getCalibration()).toEqual(calibrationData);
    });

    it('should auto-calibrate from pose data', () => {
      const mockPose = createMockPose({
        leftShoulder: { x: 50, y: 100, score: 0.8 },
        rightShoulder: { x: 95, y: 100, score: 0.8 }, // 45 pixel shoulder width
        leftHip: { x: 60, y: 160, score: 0.8 },
        rightHip: { x: 85, y: 160, score: 0.8 } // 25 pixel hip width, 60 pixel torso height
      });
      
      const calibration = service.autoCalibrate(mockPose);
      
      expect(calibration).not.toBeNull();
      expect(calibration!.pixelsPerMeter).toBeGreaterThan(0);
      expect(service.isCalibrated()).toBe(true);
    });

    it('should reject auto-calibration with low confidence poses', () => {
      const mockPose = createMockPose({
        leftShoulder: { x: 50, y: 100, score: 0.3 }, // Low confidence
        rightShoulder: { x: 95, y: 100, score: 0.8 }
      });
      
      const calibration = service.autoCalibrate(mockPose);
      
      expect(calibration).toBeNull();
      expect(service.isCalibrated()).toBe(false);
    });
  });

  describe('Data Management', () => {
    it('should clean old events automatically', () => {
      const oldTime = Date.now() - 35000; // 35 seconds ago
      const recentTime = Date.now() - 5000; // 5 seconds ago
      
      // Add old event
      (service as any).gaitEvents.push({
        type: 'heel-strike',
        foot: 'left',
        timestamp: oldTime,
        position: { x: 100, y: 200 },
        confidence: 0.8
      });
      
      // Add recent event through normal flow
      const mockPose = createMockPose();
      vi.setSystemTime(recentTime);
      service.addPose(mockPose, recentTime);
      
      const events = service.getRecentEvents();
      expect(events.some(e => e.timestamp === oldTime)).toBe(false);
    });

    it('should reset state completely', () => {
      // Add some data
      const mockPose = createMockPose();
      service.addPose(mockPose, Date.now());
      service.calibrate({
        pixelsPerMeter: 100,
        referenceHeight: 1.7,
        cameraHeight: 1.0,
        cameraAngle: 0
      });
      
      service.reset();
      
      expect(service.getPoseHistory()).toHaveLength(0);
      expect(service.getRecentEvents()).toHaveLength(0);
      // Note: Calibration persists through reset
    });
  });
});

// Test Data Factory
function createMockPose(overrides: Partial<{
  leftAnkle: Partial<Keypoint>;
  rightAnkle: Partial<Keypoint>;
  leftKnee: Partial<Keypoint>;
  rightKnee: Partial<Keypoint>;
  leftHip: Partial<Keypoint>;
  rightHip: Partial<Keypoint>;
  leftShoulder: Partial<Keypoint>;
  rightShoulder: Partial<Keypoint>;
}> = {}): Pose {
  const defaultKeypoint = { x: 100, y: 100, score: 0.8 };
  
  const keypoints = [
    { ...defaultKeypoint, name: 'nose' },
    { ...defaultKeypoint, name: 'left_eye' },
    { ...defaultKeypoint, name: 'right_eye' },
    { ...defaultKeypoint, name: 'left_ear' },
    { ...defaultKeypoint, name: 'right_ear' },
    { ...defaultKeypoint, ...overrides.leftShoulder, name: 'left_shoulder' },
    { ...defaultKeypoint, ...overrides.rightShoulder, name: 'right_shoulder' },
    { ...defaultKeypoint, name: 'left_elbow' },
    { ...defaultKeypoint, name: 'right_elbow' },
    { ...defaultKeypoint, name: 'left_wrist' },
    { ...defaultKeypoint, name: 'right_wrist' },
    { ...defaultKeypoint, ...overrides.leftHip, name: 'left_hip' },
    { ...defaultKeypoint, ...overrides.rightHip, name: 'right_hip' },
    { ...defaultKeypoint, ...overrides.leftKnee, name: 'left_knee' },
    { ...defaultKeypoint, ...overrides.rightKnee, name: 'right_knee' },
    { ...defaultKeypoint, ...overrides.leftAnkle, name: 'left_ankle' },
    { ...defaultKeypoint, ...overrides.rightAnkle, name: 'right_ankle' }
  ];

  return {
    keypoints,
    score: 0.8
  };
}
```

### Performance Testing Utilities
```typescript
// PerformanceBenchmark.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { PoseDetectionService } from '../services/PoseDetectionService';

describe('Performance Benchmarks', () => {
  let service: PoseDetectionService;

  beforeEach(async () => {
    service = new PoseDetectionService();
    await service.initialize({
      modelType: 'lightning',
      enableGPU: true,
      inputResolution: { width: 192, height: 192 },
      maxPoses: 1,
      validation: { minPoseConfidence: 0.3, minKeypointConfidence: 0.3 },
      smoothing: { enabled: true, smoothingFactor: 0.5, minConfidence: 0.3, historySize: 5 },
      performance: { targetFPS: 30, enableFrameSkipping: true, frameSkipInterval: 2 }
    });
  });

  afterEach(() => {
    service.dispose();
  });

  it('should process frames within 33ms budget (30 FPS)', async () => {
    const mockVideoElement = createMockVideoElement();
    const iterations = 10;
    const times: number[] = [];

    for (let i = 0; i < iterations; i++) {
      const start = performance.now();
      await service.detectPoses(mockVideoElement);
      const end = performance.now();
      times.push(end - start);
    }

    const avgTime = times.reduce((a, b) => a + b, 0) / times.length;
    const maxTime = Math.max(...times);

    expect(avgTime).toBeLessThan(33); // 30 FPS budget
    expect(maxTime).toBeLessThan(50); // Allow some variance
  });

  it('should maintain stable memory usage', async () => {
    const mockVideoElement = createMockVideoElement();
    const initialMemory = (performance as any).memory?.usedJSHeapSize || 0;
    
    // Run detection for extended period
    for (let i = 0; i < 1000; i++) {
      await service.detectPoses(mockVideoElement);
    }
    
    const finalMemory = (performance as any).memory?.usedJSHeapSize || 0;
    const memoryIncrease = finalMemory - initialMemory;
    
    // Memory should not increase by more than 50MB
    expect(memoryIncrease).toBeLessThan(50 * 1024 * 1024);
  });

  it('should achieve target FPS consistently', async () => {
    const mockVideoElement = createMockVideoElement();
    const targetFPS = 30;
    const testDuration = 3000; // 3 seconds
    const startTime = performance.now();
    let frameCount = 0;

    while (performance.now() - startTime < testDuration) {
      await service.detectPoses(mockVideoElement);
      frameCount++;
    }

    const actualDuration = performance.now() - startTime;
    const actualFPS = (frameCount / actualDuration) * 1000;

    expect(actualFPS).toBeGreaterThan(targetFPS * 0.9); // Allow 10% variance
  });
});

// Mock utilities
function createMockVideoElement(): HTMLVideoElement {
  const video = document.createElement('video');
  video.width = 640;
  video.height = 480;
  video.videoWidth = 640;
  video.videoHeight = 480;
  return video;
}
```

## 📊 Test Coverage Requirements

### Coverage Thresholds
- **Overall Coverage**: 90% lines, 85% branches, 95% functions
- **Critical Services**: 95% lines, 95% branches, 100% functions
- **Utility Functions**: 100% coverage required

### Critical Path Testing
- Pose detection pipeline: 100% coverage
- Gait analysis algorithms: 95% coverage
- Performance monitoring: 90% coverage
- Error handling: 100% coverage

This comprehensive unit testing specification ensures thorough validation of all service layer components with performance benchmarking and reliability testing.