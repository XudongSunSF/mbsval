Layer Architecture Overview
```cpp
//==============================================================================
// PRESENTATION LAYER - User Interface & Input Parsing
//==============================================================================

class PresentationLayer {
public:
    void initialize();
    void shutdown();
    
    // Parse CLI and produce simple DTO
    RequestDTO parseCLI(int argc, char* argv[]);
    RequestDTO parseConfig(const std::string& configFile);
    
    // Display results to user
    void displayResults(const UniversalResult& result);
};

struct RequestDTO {
    std::map<std::string, std::string> arguments;
    std::map<std::string, std::string> configuration;
    std::string sourceInterface;  // "cli", "config", etc.
};


//==============================================================================
// APPLICATION LAYER - Request Coordination & Orchestration Entry
//==============================================================================

class ApplicationLayer {
public:
    ApplicationLayer(std::shared_ptr<ModelSuiteLayer> modelSuiteLayer);
    
    // Build and validate request, then route to Model Suite Layer
    UniversalResult processRequest(const RequestDTO& dto);
    
private:
    RequestBuilder requestBuilder_;
    std::shared_ptr<ModelSuiteLayer> modelSuiteLayer_;
};

class RequestBuilder {
public:
    RequestBuilder(std::shared_ptr<SuiteRegistry> registry);
    
    // Construct domain request from DTO
    ValuationRequest buildRequest(const RequestDTO& dto);
    
    // Validate request semantics
    ValidationResult validateRequest(const ValuationRequest& request);
    
private:
    std::shared_ptr<SuiteRegistry> registry_;
};

struct ValuationRequest {
    std::string portfolioSource;      // Path or embedded indicator
    std::vector<std::string> suites;  // Suites to execute
    std::map<std::string, Parameter> parameters;
    Date effectiveDate;
    SessionConfig sessionConfig;      // For dev sessions vs. EOD runs
};


//==============================================================================
// MODEL SUITE LAYER - Business Logic & Domain Operations
//==============================================================================

// Entry point to Model Suite Layer
class ModelSuiteLayer {
public:
    ModelSuiteLayer(
        std::shared_ptr<WorkflowOrchestrator> orchestrator,
        std::shared_ptr<DataAccessLayer> dataAccess
    );
    
    UniversalResult execute(const ValuationRequest& request);
    
private:
    std::shared_ptr<WorkflowOrchestrator> orchestrator_;
    std::shared_ptr<DataAccessLayer> dataAccess_;
    std::shared_ptr<CommonPortfolioEnricher> enricher_;
};

//------------------------------------------------------------------------------
// Workflow Orchestrator - Coordinates Suite Execution
//------------------------------------------------------------------------------

class WorkflowOrchestrator {
public:
    WorkflowOrchestrator(std::shared_ptr<SuiteRegistry> registry);
    
    // Execute workflow based on dependency graph
    UniversalResult executeWorkflow(
        const Portfolio& portfolio,
        const std::vector<std::string>& requestedSuites,
        const ExecutionPlan& plan
    );
    
private:
    ExecutionPlan buildExecutionPlan(const std::vector<std::string>& suites);
    ExecutionContext prepareContext(const SuiteInfo& suite);
    
    std::shared_ptr<SuiteRegistry> registry_;
    std::shared_ptr<IntermediateResultStore> resultStore_;
};

struct ExecutionPlan {
    std::vector<SuiteInfo> executionOrder;  // Topologically sorted
    DependencyGraph dependencyGraph;
};

//------------------------------------------------------------------------------
// Suite Registry - Authoritative Suite Metadata
//------------------------------------------------------------------------------

class SuiteRegistry {
public:
    void registerSuite(const SuiteMetadata& metadata);
    
    std::shared_ptr<SuiteExecutor> getSuiteExecutor(
        const std::string& name, 
        const std::string& version
    );
    
    SuiteMetadata getMetadata(const std::string& name) const;
    std::vector<std::string> getDependencies(const std::string& name) const;
    
private:
    std::map<std::string, SuiteMetadata> registry_;
    std::map<std::string, std::shared_ptr<SuiteExecutor>> executors_;
};

struct SuiteMetadata {
    std::string name;
    std::string version;
    std::vector<std::string> dependencies;
    InputOutputContract contract;
};

//------------------------------------------------------------------------------
// Suite Executor Interface - All Suites Implement This
//------------------------------------------------------------------------------

class SuiteExecutor {
public:
    virtual ~SuiteExecutor() = default;
    
    // Main execution method
    virtual SuiteResult execute(const ExecutionContext& context) = 0;
    
    // Validation before execution
    virtual void validate(const ExecutionContext& context) const = 0;
    
    // Suite metadata
    virtual SuiteMetadata getMetadata() const = 0;
};

//------------------------------------------------------------------------------
// Execution Context - Input to Suite Execution
//------------------------------------------------------------------------------

struct ExecutionContext {
    Portfolio portfolio;  // Enriched portfolio
    std::map<std::string, Parameter> parameters;
    
    // Dependency outputs (immutable)
    DependencyOutputs dependencyOutputs;
    
    // Helper for type-safe access
    template<typename T>
    std::shared_ptr<const T> getDependency(const std::string& suiteName) const {
        return dependencyOutputs.get<T>(suiteName);
    }
    
    std::shared_ptr<IntermediateResultStore> resultStore;
    Date effectiveDate;
};

class DependencyOutputs {
public:
    template<typename T>
    void set(const std::string& suite, std::shared_ptr<const T> output);
    
    template<typename T>
    std::shared_ptr<const T> get(const std::string& suite) const;
    
private:
    std::map<std::string, std::shared_ptr<const void>> outputs_;
};

//------------------------------------------------------------------------------
// Individual Model Suites - Concrete Implementations
//------------------------------------------------------------------------------

class BehavioralSuite : public SuiteExecutor {
public:
    SuiteResult execute(const ExecutionContext& context) override;
    void validate(const ExecutionContext& context) const override;
    SuiteMetadata getMetadata() const override;
    
private:
    std::shared_ptr<PrepaymentModel> prepaymentModel_;
    std::shared_ptr<DefaultModel> defaultModel_;
    std::shared_ptr<SeverityModel> severityModel_;
};

class PricingSuite : public SuiteExecutor {
public:
    SuiteResult execute(const ExecutionContext& context) override;
    void validate(const ExecutionContext& context) const override;
    SuiteMetadata getMetadata() const override;
    
private:
    std::shared_ptr<OASPricingModel> oasModel_;
    std::shared_ptr<YieldModel> yieldModel_;
};

class CashFlowSuite : public SuiteExecutor {
public:
    SuiteResult execute(const ExecutionContext& context) override;
    void validate(const ExecutionContext& context) const override;
    SuiteMetadata getMetadata() const override;
    
private:
    std::shared_ptr<CashFlowProjectionModel> cfModel_;
};

//------------------------------------------------------------------------------
// Result Objects - Immutable Value Objects
//------------------------------------------------------------------------------

// Universal result containing all suite outputs
struct UniversalResult {
    // Suite-specific results (immutable via const shared_ptr)
    std::shared_ptr<const BehavioralSuiteResult> behavioralResult;
    std::shared_ptr<const PricingSuiteResult> pricingResult;
    std::shared_ptr<const CashFlowSuiteResult> cashFlowResult;
    
    ExecutionMetadata metadata;
};

// Individual suite result (immutable value object)
class BehavioralSuiteResult {
public:
    BehavioralSuiteResult(
        std::vector<double> prepayRates,
        std::vector<double> defaultRates,
        std::vector<double> severityRates
    );
    
    // Const accessors only (immutable)
    const std::vector<double>& getPrepaymentRates() const;
    const std::vector<double>& getDefaultRates() const;
    const std::vector<double>& getSeverityRates() const;
    
private:
    const std::vector<double> prepaymentRates_;
    const std::vector<double> defaultRates_;
    const std::vector<double> severityRates_;
};

class PricingSuiteResult {
public:
    PricingSuiteResult(
        std::vector<double> prices,
        std::vector<double> spreads,
        std::vector<double> durations
    );
    
    const std::vector<double>& getPrices() const;
    const std::vector<double>& getSpreads() const;
    const std::vector<double>& getDurations() const;
    
private:
    const std::vector<double> prices_;
    const std::vector<double> spreads_;
    const std::vector<double> durations_;
};

//------------------------------------------------------------------------------
// Common Domain Services
//------------------------------------------------------------------------------

class CommonPortfolioEnricher {
public:
    // Universal portfolio enrichment
    Portfolio enrich(const Portfolio& rawPortfolio, Date effectiveDate);
};

class IntermediateResultStore {
public:
    // Store intermediate results (const after storage)
    template<typename T>
    void store(const std::string& key, T&& value);
    
    // Retrieve intermediate results (const access)
    template<typename T>
    std::shared_ptr<const T> get(const std::string& key) const;
    
private:
    std::map<std::string, std::shared_ptr<const void>> store_;
};


//==============================================================================
// DATA ACCESS LAYER - I/O Operations
//==============================================================================

class DataAccessLayer {
public:
    // Portfolio loading
    Portfolio loadPortfolio(const std::string& source);
    
    // Market data retrieval
    MarketData loadMarketData(Date effectiveDate);
    
    // Model parameters
    ModelParameters loadParameters(const std::string& modelName);
    
    // Result persistence
    void saveResults(const UniversalResult& results, const std::string& destination);
    
private:
    std::shared_ptr<CacheLayer> cache_;
};


//==============================================================================
// CACHE LAYER - Performance Optimization
//==============================================================================

class CacheLayer {
public:
    // Multi-level cache (L1: memory, L2: Redis, L3: persistent)
    template<typename T>
    std::optional<T> get(const std::string& key);
    
    template<typename T>
    void put(const std::string& key, const T& value, Duration ttl);
    
    void invalidate(const std::string& key);
    
private:
    std::shared_ptr<L1Cache> l1_;  // In-memory
    std::shared_ptr<L2Cache> l2_;  // Redis/shared
    std::shared_ptr<L3Cache> l3_;  // Persistent
};


//==============================================================================
// SESSION MANAGEMENT - For Model Development Use Case
//==============================================================================

class SessionManager {
public:
    // Get or create development session
    std::shared_ptr<ModelDevelopmentSession> getOrCreateSession(
        const std::string& sessionId
    );
    
    // Cleanup expired sessions
    void cleanupExpiredSessions();
    
private:
    std::map<std::string, std::shared_ptr<ModelDevelopmentSession>> sessions_;
    std::chrono::minutes idleTimeout_{30};
};

class ModelDevelopmentSession {
public:
    // Process with delta optimization
    UniversalResult processRequest(const ValuationRequest& request);
    
    bool isExpired() const;
    
private:
    std::string sessionId_;
    std::chrono::steady_clock::time_point lastActivity_;
    
    // Cached objects (lifetime = session)
    std::shared_ptr<Portfolio> cachedPortfolio_;
    std::map<std::string, std::shared_ptr<Model>> loadedModels_;
    std::shared_ptr<UniversalResult> baselineResults_;
    
    DeltaChangeTracker deltaTracker_;
};


//==============================================================================
// MAIN ENTRY POINTS
//==============================================================================

// For model development (interactive sessions)
class InteractiveApplication {
public:
    void run() {
        PresentationLayer presentation;
        presentation.initialize();
        
        while (true) {
            auto dto = presentation.parseCLI(/* ... */);
            auto session = sessionManager_.getOrCreateSession(dto.sessionId);
            auto result = session->processRequest(/* ... */);
            presentation.displayResults(result);
        }
    }
    
private:
    SessionManager sessionManager_;
};

// For production EOD runs (batch processing)
class EODApplication {
public:
    int run(int argc, char* argv[]) {
        PresentationLayer presentation;
        auto dto = presentation.parseCLI(argc, argv);
        
        ApplicationLayer application(modelSuiteLayer_);
        auto result = application.processRequest(dto);
        
        presentation.displayResults(result);
        return 0;  // Process exits, OS cleans up all memory
    }
    
private:
    std::shared_ptr<ModelSuiteLayer> modelSuiteLayer_;
};
```
Key Design Highlights
``` cpp
//==============================================================================
// IMMUTABILITY PATTERN
//==============================================================================

// Results are immutable after creation
UniversalResult result;

result.behavioralResult = std::make_shared<const BehavioralSuiteResult>(
            behavioralSuite_->execute(context));

// Downstream suites receive const references
auto prepayRates = result.behaviralResult->getPrepaymentRates();  // const&

// Compile-time guarantee: cannot modify upstream results
// prepayRates[0] = 0.5;  // COMPILE ERROR

```

//==============================================================================
// ORCHESTRATOR-MEDIATED COMMUNICATION
//==============================================================================

// Suites never communicate directly
// Orchestrator manages all data flow

WorkflowOrchestrator:
  1. Execute Behavioral Suite → store result
  2. Pass result to Pricing Suite (as const)
  3. Execute Pricing Suite → store result
  4. Aggregate into UniversalResult


//==============================================================================
// EXECUTION FLOW
//==============================================================================

User Request
    ↓
[Presentation Layer] - Parse CLI/Config → RequestDTO
    ↓
[Application Layer] - Build Request → ValuationRequest
    ↓                - Validate Request
    ↓
[Model Suite Layer] - Load Portfolio
    |                - Enrich Portfolio
    |                - Build Execution Plan (dependency graph)
    |                
    ├─→ [Behavioral Suite] execute() → BehavioralSuiteResult (const)
    |       ↓
    ├─→ [Pricing Suite] execute(behavioral results) → PricingSuiteResult (const)
    |       ↓
    └─→ [Cash Flow Suite] execute(behavioral results) → CashFlowSuiteResult (const)
    ↓
UniversalResult (aggregated, all results immutable)
    ↓
[Application Layer] - Format Results
    ↓
[Presentation Layer] - Display to User


Adding GPU Executor Support

```
✓ Presentation Layer          - No changes
✓ Application Layer           - No changes
✓ WorkflowOrchestrator       - No changes (doesn't care about execution strategy)
✓ SuiteRegistry              - No changes
✓ SuiteExecutor Interface    - No changes (same execute() contract)
✓ Data Access Layer          - No changes
✓ Cache Layer                - No changes
✓ UniversalResult structure  - No changes
✓ ExecutionContext           - No changes
```

GPU execution is an internal implementation detail of individual model suites. The orchestrator doesn't need to know how a suite executes, only what it produces.

What Changes (Localized to Model Suites)
1. Suite Implementation - Internal Execution Strategy

```cpp
//==============================================================================
// Model Suite with GPU Support
//==============================================================================

class BehavioralSuite : public SuiteExecutor {
public:
    BehavioralSuite(ExecutionDevice device = ExecutionDevice::CPU);
    
    SuiteResult execute(const ExecutionContext& context) override {
        // Same interface, different implementation
        
        if (device_ == ExecutionDevice::GPU) {
            return executeOnGPU(context);
        } else {
            return executeOnCPU(context);
        }
    }
    
private:
    ExecutionDevice device_;
    
    SuiteResult executeOnCPU(const ExecutionContext& context);
    SuiteResult executeOnGPU(const ExecutionContext& context);
    
    // GPU-specific resources
    std::unique_ptr<GPUContext> gpuContext_;
    std::shared_ptr<PrepaymentModelGPU> prepaymentModelGPU_;
    std::shared_ptr<DefaultModelGPU> defaultModelGPU_;
    
    // CPU resources (keep for fallback)
    std::shared_ptr<PrepaymentModel> prepaymentModelCPU_;
    std::shared_ptr<DefaultModel> defaultModelCPU_;
};

enum class ExecutionDevice {
    CPU,
    GPU,
    AUTO  // Automatically select based on data size/availability
};
```

Key Point: The execute() interface remains identical. GPU is an internal implementation choice.

2. Configuration - Specify Execution Device
```cpp
//==============================================================================
// Suite Metadata Extension (Optional)
//==============================================================================

struct SuiteMetadata {
    std::string name;
    std::string version;
    std::vector<std::string> dependencies;
    InputOutputContract contract;
    
    // Optional: GPU capability metadata
    ExecutionCapabilities capabilities;
};

struct ExecutionCapabilities {
    bool supportsGPU;
    bool supportsCPU;
    size_t minGPUMemoryMB;  // Minimum GPU memory required
    std::vector<std::string> gpuArchitectures;  // "CUDA", "ROCm", "Metal"
};

//==============================================================================
// Suite Metadata Extension (Optional)
//==============================================================================

struct SuiteMetadata {
    std::string name;
    std::string version;
    std::vector<std::string> dependencies;
    InputOutputContract contract;
    
    // Optional: GPU capability metadata
    ExecutionCapabilities capabilities;
};

struct ExecutionCapabilities {
    bool supportsGPU;
    bool supportsCPU;
    size_t minGPUMemoryMB;  // Minimum GPU memory required
    std::vector<std::string> gpuArchitectures;  // "CUDA", "ROCm", "Metal"
};

//==============================================================================
// Request-Level Configuration
//==============================================================================

struct ValuationRequest {
    std::string portfolioSource;
    std::vector<std::string> suites;
    std::map<std::string, Parameter> parameters;
    Date effectiveDate;
    SessionConfig sessionConfig;
    
    // NEW: Execution preferences (optional)
    ExecutionConfig executionConfig;
};

struct ExecutionConfig {
    ExecutionDevice preferredDevice = ExecutionDevice::AUTO;
    
    // Per-suite overrides
    std::map<std::string, ExecutionDevice> suiteDeviceOverrides;
    // Example: {"behavioral": GPU, "pricing": CPU}
    
    // GPU resource limits
    size_t maxGPUMemoryMB = 8192;
    int gpuDeviceId = 0;  // Which GPU to use
};
```

3. Suite Registry - Device-Aware Instantiation
```cpp
//==============================================================================
// Suite Registry Creates Executors with Requested Device
//==============================================================================

class SuiteRegistry {
public:
    std::shared_ptr<SuiteExecutor> getSuiteExecutor(
        const std::string& name,
        const std::string& version,
        ExecutionDevice device = ExecutionDevice::CPU  // NEW parameter
    ) {
        auto key = makeSuiteKey(name, version, device);
        
        // Check if already instantiated
        auto it = executors_.find(key);
        if (it != executors_.end()) {
            return it->second;
        }
        
        // Create new executor with requested device
        auto executor = createExecutor(name, version, device);
        executors_[key] = executor;
        return executor;
    }
    
private:
    std::shared_ptr<SuiteExecutor> createExecutor(
        const std::string& name,
        const std::string& version,
        ExecutionDevice device
    ) {
        if (name == "behavioral") {
            return std::make_shared<BehavioralSuite>(device);
        }
        else if (name == "pricing") {
            return std::make_shared<PricingSuite>(device);
        }
        // ...
    }
    
    std::map<std::string, std::shared_ptr<SuiteExecutor>> executors_;
};
```

4. Workflow Orchestrator - Minimal Change
   ```cpp
   //==============================================================================
// Orchestrator Passes Device Preference to Registry
//==============================================================================

class WorkflowOrchestrator {
public:
    UniversalResult executeWorkflow(
        const Portfolio& portfolio,
        const std::vector<std::string>& requestedSuites,
        const ExecutionPlan& plan,
        const ExecutionConfig& executionConfig  // NEW parameter
    ) {
        UniversalResult result;
        
        for (const auto& suiteInfo : plan.executionOrder) {
            // Determine device for this suite
            ExecutionDevice device = determineDevice(
                suiteInfo.name,
                executionConfig
            );
            
            // Get executor with device preference (ONLY CHANGE)
            auto suite = registry_->getSuiteExecutor(
                suiteInfo.name,
                suiteInfo.version,
                device  // Pass device preference
            );
            
            // Everything else unchanged
            ExecutionContext context = prepareContext(suiteInfo, result);
            auto suiteResult = suite->execute(context);
            
            // Store result (same as before)
            storeSuiteResult(suiteInfo.name, suiteResult, result);
        }
        
        return result;
    }
    
private:
    ExecutionDevice determineDevice(
        const std::string& suiteName,
        const ExecutionConfig& config
    ) {
        // Check suite-specific override
        auto it = config.suiteDeviceOverrides.find(suiteName);
        if (it != config.suiteDeviceOverrides.end()) {
            return it->second;
        }
        
        // Use default preference
        if (config.preferredDevice == ExecutionDevice::AUTO) {
            return autoSelectDevice(suiteName);
        }
        
        return config.preferredDevice;
    }
    
    ExecutionDevice autoSelectDevice(const std::string& suiteName) {
        // Check if GPU available
        if (!GPUContext::isAvailable()) {
            return ExecutionDevice::CPU;
        }
        
        // Check suite capabilities
        auto metadata = registry_->getMetadata(suiteName);
        if (!metadata.capabilities.supportsGPU) {
            return ExecutionDevice::CPU;
        }
        
        // Use GPU if available and supported
        return ExecutionDevice::GPU;
    }
};
```
Implementation Within Suite
Example: Behavioral Suite with GPU Support
//==============================================================================
// Suite Internal Implementation - Where GPU Logic Lives
//==============================================================================

class BehavioralSuite : public SuiteExecutor {
private:
    SuiteResult executeOnCPU(const ExecutionContext& context) {
        // Existing CPU implementation
        auto prepayResult = prepaymentModelCPU_->calculate(
            context.portfolio,
            context.parameters
        );
        
        auto defaultResult = defaultModelCPU_->calculate(
            context.portfolio,
            context.parameters
        );
        
        return BehavioralSuiteResult(prepayResult, defaultResult);
    }
    
    SuiteResult executeOnGPU(const ExecutionContext& context) {
        // 1. Transfer portfolio data to GPU
        auto portfolioGPU = gpuContext_->transferToGPU(context.portfolio);
        auto paramsGPU = gpuContext_->transferToGPU(context.parameters);
        
        // 2. Execute models on GPU (parallel if possible)
        auto prepayResultGPU = prepaymentModelGPU_->calculate(
            portfolioGPU,
            paramsGPU
        );
        
        auto defaultResultGPU = defaultModelGPU_->calculate(
            portfolioGPU,
            paramsGPU
        );
        
        // 3. Transfer results back to CPU
        auto prepayResult = gpuContext_->transferFromGPU(prepayResultGPU);
        auto defaultResult = gpuContext_->transferFromGPU(defaultResultGPU);
        
        // 4. Return same result format (CPU or GPU, result structure identical)
        return BehavioralSuiteResult(prepayResult, defaultResult);
    }
    
    // GPU resource management
    std::unique_ptr<GPUContext> gpuContext_;
};

//==============================================================================
// GPU Context - Manages GPU Resources
//==============================================================================

class GPUContext {
public:
    static bool isAvailable();
    static size_t getAvailableMemoryMB();
    
    GPUContext(int deviceId = 0);
    ~GPUContext();  // Cleanup GPU resources
    
    // Data transfer
    template<typename T>
    GPUBuffer<T> transferToGPU(const std::vector<T>& data);
    
    template<typename T>
    std::vector<T> transferFromGPU(const GPUBuffer<T>& buffer);
    
private:
    int deviceId_;
    void* cudaStream_;  // Or equivalent for other GPU platforms
};
```

---

## Minimal Changes Summary
```
Layer                        Change Required?    Change Type
═══════════════════════════════════════════════════════════════
Presentation Layer           NO                  -
Application Layer            NO                  -
RequestBuilder               NO                  -
WorkflowOrchestrator        MINOR               Pass device config
SuiteRegistry               MINOR               Device-aware instantiation
SuiteExecutor Interface     NO                  -
UniversalResult             NO                  -
ExecutionContext            NO                  -
Data Access Layer           NO                  -
Cache Layer                 NO                  -

Individual Model Suites     YES (per suite)     Add GPU implementation
└─ BehavioralSuite          YES                 executeOnGPU()
└─ PricingSuite             YES (if needed)     executeOnGPU()
└─ CashFlowSuite            YES (if needed)     executeOnGPU()
