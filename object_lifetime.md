Object Lifetime Strategy for Both Use Cases
Use Case 1: Model Development & Tuning (Interactive/Iterative)
Objective: Minimize overhead between iterations when only parameters/configurations change by keeping expensive objects alive.
Strategy: Long-Running Process with Idle Timeout
Architecture:
┌─────────────────────────────────────────────┐
│ Model Development Session                   │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ Session Manager                     │     │
│ │ - Session ID/Token                  │     │
│ │ - Idle timeout (e.g., 30 minutes)  │     │
│ │ - Activity tracking                 │     │
│ └─────────────────────────────────────┘     │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ In-Process Object Cache             │     │
│ │ ┌─────────────────────────────┐     │     │
│ │ │ Portfolio (loaded & enriched)│    │     │
│ │ │ - Lifetime: Session         │     │     │
│ │ └─────────────────────────────┘     │     │
│ │ ┌─────────────────────────────┐     │     │
│ │ │ Loaded Models (heavy)       │     │     │
│ │ │ - Lifetime: Session         │     │     │
│ │ └─────────────────────────────┘     │     │
│ │ ┌─────────────────────────────┐     │     │
│ │ │ Market Data                 │     │     │
│ │ │ - Lifetime: Session or TTL  │     │     │
│ │ └─────────────────────────────┘     │     │
│ │ ┌─────────────────────────────┐     │     │
│ │ │ Baseline Results            │     │     │
│ │ │ - Lifetime: Session         │     │     │
│ │ └─────────────────────────────┘     │     │
│ └─────────────────────────────────────┘     │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ Iteration State                     │     │
│ │ - Previous run results              │     │
│ │ - Parameter history                 │     │
│ │ - Delta change tracking             │     │
│ └─────────────────────────────────────┘     │
└─────────────────────────────────────────────┘
Object Lifetime Rules:

Session-Scoped Objects (Lifetime: Until session timeout)

Portfolio: Load once, reuse across all iterations
Loaded Models: Heavy objects (neural nets, trees) - keep in memory
Common Enrichment Results: Expensive, portfolio-derived data
Baseline Results: First full run results for delta comparison


Iteration-Scoped Objects (Lifetime: Single iteration)

Suite Results: From current iteration only
Intermediate Results: Specific to current parameter set
Changed Parameter Objects: Only objects affected by delta


Time-Based TTL Objects (Lifetime: 5-15 minutes)

Market Data: Can be refreshed if stale
Reference Data: Lower change frequency



Session Management:
cppclass ModelDevelopmentSession {
private:
    std::string sessionId_;
    std::chrono::steady_clock::time_point lastActivity_;
    std::chrono::minutes idleTimeout_{30};
    
    // Session-scoped objects
    std::shared_ptr<Portfolio> cachedPortfolio_;
    std::unordered_map<std::string, std::shared_ptr<Model>> loadedModels_;
    std::shared_ptr<SuiteResult> baselineResults_;
    
    // Delta change tracker
    DeltaChangeTracker deltaTracker_;
    
public:
    /**
     * Process request with delta optimization
     * Returns: execution_type (full vs delta)
     */
    ExecutionResult processRequest(const ValuationRequest& request) {
        updateActivity();
        
        // Detect what changed since last run
        auto changes = deltaTracker_.detectChanges(request);
        
        if (changes.isPortfolioChanged()) {
            // Portfolio changed - need full rerun
            invalidateSessionCache();
            return executeFullRun(request);
        }
        
        if (changes.isOnlyParametersChanged()) {
            // Only parameters changed - delta optimization
            return executeDeltaRun(request, changes);
        }
        
        return executeFullRun(request);
    }
    
    bool isExpired() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::minutes>(
            now - lastActivity_
        );
        return elapsed > idleTimeout_;
    }
};
```

**Process Lifecycle:**
```
┌────────────────┐
│ First Request  │
│ (new session)  │
└───────┬────────┘
        │
        ▼
┌────────────────────────────────┐
│ 1. Create Session              │
│    - Generate session ID       │
│    - Initialize cache          │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 2. Load Heavy Objects          │
│    - Portfolio (full load)     │
│    - Common enrichment         │
│    - Model loading             │
│    - Baseline calculation      │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 3. Subsequent Requests         │
│    (same session ID)           │
│                                │
│    - Reuse cached portfolio    │
│    - Reuse loaded models       │
│    - Only reload changed params│
│    - Delta-only execution      │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 4. Idle Timeout Reached        │
│    (30 min no activity)        │
│                                │
│    - Cleanup session objects   │
│    - Release memory            │
│    - Terminate session         │
└────────────────────────────────┘
Why Long-Running Process (Not Redis):
AspectLong-Running ProcessRedis CachePerformance✓ Direct memory access, no serialization✗ Serialization overhead, network latencyObject Complexity✓ Complex C++ objects stay in memory✗ Must serialize/deserialize complex objectsMemory Efficiency✓ Native C++ memory management✗ Double memory (process + Redis)Code Complexity✓ Simpler - just keep objects alive✗ Serialization code for all cached typesModel Objects✓ Can keep ML models in memory as-is✗ Very difficult to serialize trained modelsState Management✓ Session state naturally encapsulated✗ Distributed state harder to reason about
Implementation Pattern:
cppclass DevelopmentSessionManager {
private:
    std::unordered_map<std::string, std::shared_ptr<ModelDevelopmentSession>> 
        activeSessions_;
    std::mutex sessionsMutex_;
    
public:
    std::shared_ptr<ModelDevelopmentSession> getOrCreateSession(
        const std::string& sessionId
    ) {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        auto it = activeSessions_.find(sessionId);
        if (it != activeSessions_.end() && !it->second->isExpired()) {
            return it->second;  // Reuse existing session
        }
        
        // Create new session
        auto session = std::make_shared<ModelDevelopmentSession>(sessionId);
        activeSessions_[sessionId] = session;
        return session;
    }
    
    // Background thread periodically cleans up expired sessions
    void cleanupExpiredSessions() {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        auto it = activeSessions_.begin();
        while (it != activeSessions_.end()) {
            if (it->second->isExpired()) {
                LOG(INFO) << "Cleaning up expired session: " << it->first;
                it = activeSessions_.erase(it);
            } else {
                ++it;
            }
        }
    }
};
```

---

### Use Case 2: Production EOD Valuation (Batch Processing)

**Objective:** Reliable, one-shot execution with full cleanup. No state persistence between runs.

**Strategy: Process Per Run (Ephemeral)**

**Architecture:**
```
┌─────────────────────────────────────────────┐
│ EOD Valuation Run (Single Process)         │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ Process Lifecycle                   │     │
│ │ 1. Start → 2. Execute → 3. Cleanup  │     │
│ └─────────────────────────────────────┘     │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ Run-Scoped Objects                  │     │
│ │ - All objects lifetime = run        │     │
│ │ - No cross-run caching              │     │
│ │ - Clean slate every time            │     │
│ └─────────────────────────────────────┘     │
│                                             │
│ ┌─────────────────────────────────────┐     │
│ │ Resource Management                 │     │
│ │ - RAII for all resources            │     │
│ │ - Explicit cleanup on exit          │     │
│ │ - Memory leak detection             │     │
│ └─────────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

**Object Lifetime Rules:**

1. **All Objects: Run-Scoped (Create → Use → Destroy)**
   - Portfolio: Load → Process → Release
   - Models: Load → Execute → Unload
   - Results: Calculate → Persist → Discard
   - Market Data: Fetch → Use → Release

2. **No Persistent State Between Runs**
   - Each EOD run is independent
   - Previous run results in database/files, not memory
   - Fresh process with clean memory space

3. **External Cache Usage (Optional)**
   - Use Redis/database cache ONLY for:
     - Pre-loaded market data (populated before EOD run starts)
     - Static reference data
   - NOT for keeping run state alive

**Process Lifecycle:**
```
┌────────────────┐
│ EOD Scheduler  │
│ (cron/airflow) │
└───────┬────────┘
        │
        ▼
┌────────────────────────────────┐
│ 1. Start Fresh Process         │
│    - Clean memory space        │
│    - Load configuration        │
│    - Initialize components     │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 2. Load Everything             │
│    - Portfolio from DB/file    │
│    - EOD market data           │
│    - Model parameters          │
│    - All models (cold start)   │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 3. Execute Workflow            │
│    - Full workflow execution   │
│    - All suites, all models    │
│    - Complete calculation      │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 4. Persist Results             │
│    - Write to database         │
│    - Generate reports          │
│    - Create audit trail        │
└────────┬───────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│ 5. Cleanup & Exit              │
│    - Explicit resource cleanup │
│    - Memory validation         │
│    - Log completion            │
│    - Process terminates        │
└────────────────────────────────┘
Implementation Pattern:
cppclass EODValuationRunner {
public:
    /**
     * Single-shot execution - no state preservation
     */
    int runEODValuation(const EODConfig& config) {
        try {
            // 1. Initialize (RAII ensures cleanup)
            auto portfolio = loadPortfolio(config.portfolioSource);
            auto marketData = loadMarketData(config.effectiveDate);
            auto models = loadAllModels(config.modelVersions);
            
            // 2. Execute full workflow
            WorkflowOrchestrator orchestrator(portfolio, marketData, models);
            auto results = orchestrator.executeFullWorkflow(config.suites);
            
            // 3. Persist results
            persistResults(results, config.outputDestination);
            
            // 4. Cleanup is automatic (RAII destructors)
            return 0;  // Success
            
        } catch (const std::exception& e) {
            LOG(ERROR) << "EOD run failed: " << e.what();
            return 1;  // Failure
        }
        // All objects destroyed here automatically
    }
};

// Main entry point for EOD process
int main(int argc, char* argv[]) {
    auto config = parseEODConfig(argc, argv);
    
    EODValuationRunner runner;
    int exitCode = runner.runEODValuation(config);
    
    // Process exits, OS reclaims all memory
    return exitCode;
}
```

**Why Ephemeral Process (Not Long-Running):**

| Aspect | Ephemeral Process | Long-Running Process |
|--------|-------------------|---------------------|
| **Reliability** | ✓ Fresh start, no accumulated state bugs | ✗ Memory leaks accumulate over time |
| **Resource Management** | ✓ OS cleanup guaranteed | ✗ Must implement perfect cleanup |
| **Debugging** | ✓ Clear start/end, reproducible | ✗ State from previous runs may interfere |
| **Memory Leaks** | ✓ Process exit reclaims everything | ✗ Small leaks become big problems |
| **Monitoring** | ✓ Simple metrics (process start/end) | ✗ Complex state tracking needed |
| **Failure Recovery** | ✓ Just restart process | ✗ Must reset state correctly |

---

### Comparison & Recommendations

| Aspect | Model Development | Production EOD |
|--------|------------------|----------------|
| **Process Model** | Long-running with idle timeout | Ephemeral per-run |
| **Session Management** | Required (session ID) | Not needed |
| **Object Caching** | In-process memory cache | No caching between runs |
| **Idle Timeout** | 15-30 minutes | N/A (process exits immediately) |
| **State Persistence** | In-memory for session duration | No cross-run persistence |
| **Cleanup Strategy** | Timeout-based cleanup | Immediate cleanup on exit |
| **External Cache Usage** | Minimal (only for reference data) | Optional (market data pre-loading) |

### Hybrid Consideration: Redis for Cross-Process Sharing

**When Redis Makes Sense (Both Use Cases):**
```
┌──────────────────────────────────────────┐
│ Redis (Machine-Level Cache)              │
│                                          │
│ ┌────────────────────────────────┐       │
│ │ Shareable Static Data          │       │
│ │ - Market data (curves, indices)│       │
│ │ - Reference data (pool chars)  │       │
│ │ - Model parameters (read-only) │       │
│ └────────────────────────────────┘       │
│                                          │
│ TTL: Hours to days                       │
└──────────────────────────────────────────┘
         ↑                    ↑
         │                    │
    ┌────┴─────┐         ┌────┴─────┐
    │ Dev      │         │ EOD      │
    │ Session  │         │ Process  │
    └──────────┘         └──────────┘
