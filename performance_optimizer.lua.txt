-- performance_optimizer.lua
-- High-end Lua script for real-time performance optimization
-- Features: Kalman filter, exponential moving average, memory pooling,
-- real-time data simulation with compression, adaptive performance scaling,
-- debug toggles, and benchmarking.
-- Designed for modular import into main Lua scripts.

local performance_optimizer = {}

-- Debug toggle
local DEBUG = false

-- Utility function for debug printing
local function debug_print(...)
    if DEBUG then
        print("[DEBUG]", ...)
    end
end

-- Benchmarking utility
local Benchmark = {}
Benchmark.__index = Benchmark

function Benchmark:new(name)
    local obj = {
        name = name or "Benchmark",
        start_time = 0,
        elapsed = 0,
        count = 0
    }
    setmetatable(obj, self)
    return obj
end

function Benchmark:start()
    self.start_time = os.clock()
end

function Benchmark:stop()
    local end_time = os.clock()
    self.elapsed = self.elapsed + (end_time - self.start_time)
    self.count = self.count + 1
end

function Benchmark:get_average()
    if self.count == 0 then return 0 end
    return self.elapsed / self.count
end

function Benchmark:reset()
    self.elapsed = 0
    self.count = 0
end

performance_optimizer.Benchmark = Benchmark

-- Kalman Filter module
local KalmanFilter = {}
KalmanFilter.__index = KalmanFilter

function KalmanFilter:new(R, Q, initial_value)
    -- R: measurement noise covariance
    -- Q: process noise covariance
    -- initial_value: initial estimate
    local obj = {
        R = R or 0.01,
        Q = Q or 0.0001,
        x = initial_value or 0, -- initial estimate
        P = 1, -- initial estimation error covariance
        K = 0 -- initial Kalman gain
    }
    setmetatable(obj, self)
    return obj
end

function KalmanFilter:update(z)
    -- z: measurement
    -- Prediction update
    self.P = self.P + self.Q

    -- Measurement update
    self.K = self.P / (self.P + self.R)
    self.x = self.x + self.K * (z - self.x)
    self.P = (1 - self.K) * self.P

    debug_print("KalmanFilter update:", z, "estimate:", self.x, "gain:", self.K)
    return self.x
end

performance_optimizer.KalmanFilter = KalmanFilter

-- Exponential Moving Average (EMA) module for delta time smoothing
local EMA = {}
EMA.__index = EMA

function EMA:new(alpha, initial_value)
    -- alpha: smoothing factor (0 < alpha <= 1)
    -- initial_value: initial average value
    local obj = {
        alpha = alpha or 0.1,
        value = initial_value or 0,
        initialized = false
    }
    setmetatable(obj, self)
    return obj
end

function EMA:update(new_value)
    if not self.initialized then
        self.value = new_value
        self.initialized = true
    else
        self.value = self.alpha * new_value + (1 - self.alpha) * self.value
    end
    debug_print("EMA update:", new_value, "smoothed:", self.value)
    return self.value
end

performance_optimizer.EMA = EMA

-- Memory Pool module to avoid garbage collection spikes
local MemoryPool = {}
MemoryPool.__index = MemoryPool

function MemoryPool:new(create_func, initial_size)
    -- create_func: function to create new objects
    -- initial_size: initial pool size
    local obj = {
        pool = {},
        create_func = create_func,
        size = initial_size or 100,
        used = 0
    }
    setmetatable(obj, self)
    -- Pre-allocate pool
    for i = 1, obj.size do
        obj.pool[i] = create_func()
    end
    return obj
end

function MemoryPool:acquire()
    if self.used < self.size then
        self.used = self.used + 1
        local obj = self.pool[self.used]
        debug_print("MemoryPool acquire: reused object", self.used)
        return obj
    else
        -- Pool exhausted, create new object (avoid if possible)
        local obj = self.create_func()
        debug_print("MemoryPool acquire: created new object beyond pool")
        return obj
    end
end

function MemoryPool:release(obj)
    if self.used > 0 then
        self.pool[self.used] = obj
        self.used = self.used - 1
        debug_print("MemoryPool release: returned object", self.used + 1)
    else
        -- Pool empty, discard object
        debug_print("MemoryPool release: pool empty, discarded object")
    end
end

performance_optimizer.MemoryPool = MemoryPool

-- Data Simulator module for real-time data gathering and sending
local DataSimulator = {}
DataSimulator.__index = DataSimulator

function DataSimulator:new()
    local obj = {
        last_data = nil,
        pool = MemoryPool:new(function() return {value=0} end, 50),
        compression_enabled = true,
        debug = false
    }
    setmetatable(obj, self)
    return obj
end

-- Simple delta encoding compression
function DataSimulator:compress(data)
    if not self.last_data then
        self.last_data = data
        return data
    end
    local compressed = {}
    for k,v in pairs(data) do
        local delta = v - (self.last_data[k] or 0)
        compressed[k] = delta
    end
    self.last_data = data
    if self.debug then
        debug_print("DataSimulator compress:", compressed)
    end
    return compressed
end

-- Simulate data gathering
function DataSimulator:gather_data()
    -- Simulate FPS and latency with some noise
    local fps = 60 + (math.random() - 0.5) * 10
    local latency = 30 + (math.random() - 0.5) * 10
    return {fps = fps, latency = latency}
end

-- Simulate sending data (no actual network, just placeholder)
function DataSimulator:send_data(data)
    if self.debug then
        debug_print("DataSimulator send_data:", data)
    end
    -- Placeholder for sending data logic
end

-- Full cycle: gather, compress, send
function DataSimulator:cycle()
    local data = self:gather_data()
    local compressed = self.compression_enabled and self:compress(data) or data
    self:send_data(compressed)
    return compressed
end

performance_optimizer.DataSimulator = DataSimulator

-- Adaptive Performance module for scaling processing based on performance
local AdaptivePerformance = {}
AdaptivePerformance.__index = AdaptivePerformance

function AdaptivePerformance:new()
    local obj = {
        fps_threshold = 50, -- FPS dip threshold
        ram_threshold = 100 * 1024 * 1024, -- 100 MB RAM pressure threshold (example)
        processing_scale = 1.0,
        debug = false,
        last_fps = 60,
        last_ram = 0
    }
    setmetatable(obj, self)
    return obj
end

-- Dummy function to get current RAM usage (to be replaced with actual system call if available)
function AdaptivePerformance:get_ram_usage()
    -- Placeholder: simulate RAM usage with some noise
    local ram = 80 * 1024 * 1024 + (math.random() - 0.5) * 40 * 1024 * 1024
    return ram
end

function AdaptivePerformance:update(fps)
    local ram = self:get_ram_usage()
    self.last_fps = fps
    self.last_ram = ram

    if fps < self.fps_threshold or ram > self.ram_threshold then
        -- Scale down processing
        self.processing_scale = math.max(0.1, self.processing_scale - 0.1)
        if self.debug then
            debug_print(string.format("AdaptivePerformance: Scaling down processing to %.2f (FPS: %.2f, RAM: %.2f MB)", self.processing_scale, fps, ram / (1024*1024)))
        end
    else
        -- Scale up processing gradually
        self.processing_scale = math.min(1.0, self.processing_scale + 0.05)
        if self.debug then
            debug_print(string.format("AdaptivePerformance: Scaling up processing to %.2f (FPS: %.2f, RAM: %.2f MB)", self.processing_scale, fps, ram / (1024*1024)))
        end
    end

    return self.processing_scale
end

performance_optimizer.AdaptivePerformance = AdaptivePerformance

-- Main integration example
function performance_optimizer.run_simulation(iterations, debug_mode)
    DEBUG = debug_mode or false

    local kalman_fps = KalmanFilter:new(0.1, 0.01, 60)
    local kalman_latency = KalmanFilter:new(0.1, 0.01, 30)
    local ema_delta = EMA:new(0.1, 0)
    local data_sim = DataSimulator:new()
    data_sim.debug = DEBUG
    local adaptive_perf = AdaptivePerformance:new()
    adaptive_perf.debug = DEBUG

    local bench_kalman = Benchmark:new("KalmanFilter")
    local bench_ema = Benchmark:new("EMA")
    local bench_data = Benchmark:new("DataSimulator")
    local bench_adaptive = Benchmark:new("AdaptivePerformance")

    for i = 1, iterations do
        -- Simulate delta time (time between frames)
        local raw_delta = 1 / (math.random(45, 75)) -- simulate variable FPS between 45 and 75
        local smoothed_delta = ema_delta:update(raw_delta)

        bench_kalman:start()
        local smoothed_fps = kalman_fps:update(1 / smoothed_delta)
        local smoothed_latency = kalman_latency:update(math.random(20, 40))
        bench_kalman:stop()

        bench_data:start()
        local compressed_data = data_sim:cycle()
        bench_data:stop()

        bench_adaptive:start()
        local scale = adaptive_perf:update(smoothed_fps)
        bench_adaptive:stop()

        bench_ema:start()
        -- Additional EMA updates or processing can be done here if needed
        bench_ema:stop()

        if DEBUG then
            print(string.format("Iteration %d: FPS=%.2f, Latency=%.2f, Scale=%.2f", i, smoothed_fps, smoothed_latency, scale))
        end
    end

    if DEBUG then
        print("Benchmark results (average seconds per call):")
        print(string.format("KalmanFilter: %.6f", bench_kalman:get_average()))
        print(string.format("EMA: %.6f", bench_ema:get_average()))
        print(string.format("DataSimulator: %.6f", bench_data:get_average()))
        print(string.format("AdaptivePerformance: %.6f", bench_adaptive:get_average()))
    end
end

return performance_optimizer
