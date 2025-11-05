Model Development Application - Session Management & Idle Shutdown
Architecture Overview

//==============================================================================
// HIGH-LEVEL ARCHITECTURE
//==============================================================================

┌─────────────────────────────────────────────────────────────┐
│ InteractiveApplication (main process)                       │
│                                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │ SessionManager                               │           │
│  │ - Creates/retrieves sessions                 │           │
│  │ - Monitors idle timeouts (background thread) │           │
│  │ - Cleans up expired sessions                 │           │
│  └──────────────────────────────────────────────┘           │
│                    │                                        │
│                    │ manages multiple                       │
│                    ▼                                        │
│  ┌──────────────────────────────────────────────┐           │
│  │ ModelDevelopmentSession (per user/session)   │           │
│  │ - Tracks request history                     │           │
│  │ - Caches heavy objects                       │           │
│  │ - Detects delta changes                      │           │
│  │ - Updates last activity timestamp            │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘

1. Session Manager - Multi-Session Orchestration

//==============================================================================
// SESSION MANAGER - Manages Multiple Development Sessions
//==============================================================================

class SessionManager {
public:
    SessionManager(std::chrono::minutes idleTimeout = std::chrono::minutes(30))
        : idleTimeout_(idleTimeout), running_(false) {}
    
    ~SessionManager() {
        stop();
    }
    
    // Start background monitoring
    void start() {
        running_ = true;
        cleanupThread_ = std::thread(&SessionManager::cleanupLoop, this);
    }
    
    // Stop background monitoring and cleanup all sessions
    void stop() {
        running_ = false;
        if (cleanupThread_.joinable()) {
            cleanupThread_.join();
        }
        cleanupAllSessions();
    }
    
    // Get or create a session for the given session ID
    std::shared_ptr<ModelDevelopmentSession> getOrCreateSession(
        const std::string& sessionId
    ) {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        // Check if session exists and is not expired
        auto it = activeSessions_.find(sessionId);
        if (it != activeSessions_.end()) {
            if (!it->second->isExpired()) {
                LOG(INFO) << "Reusing existing session: " << sessionId;
                return it->second;
            } else {
                LOG(INFO) << "Session expired, creating new: " << sessionId;
                activeSessions_.erase(it);
            }
        }
        
        // Create new session
        LOG(INFO) << "Creating new session: " << sessionId;
        auto session = std::make_shared<ModelDevelopmentSession>(
            sessionId,
            idleTimeout_
        );
        activeSessions_[sessionId] = session;
        
        return session;
    }
    
    // Get session statistics
    SessionStatistics getStatistics() const {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        SessionStatistics stats;
        stats.totalSessions = activeSessions_.size();
        
        for (const auto& [id, session] : activeSessions_) {
            if (session->isExpired()) {
                stats.expiredSessions++;
            } else {
                stats.activeSessions++;
            }
            stats.totalRequests += session->getRequestCount();
        }
        
        return stats;
    }
    
private:
    // Background thread that periodically cleans up expired sessions
    void cleanupLoop() {
        while (running_) {
            // Sleep for cleanup interval (e.g., every 5 minutes)
            std::this_thread::sleep_for(std::chrono::minutes(5));
            
            if (!running_) break;
            
            cleanupExpiredSessions();
        }
    }
    
    // Remove expired sessions
    void cleanupExpiredSessions() {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        auto it = activeSessions_.begin();
        while (it != activeSessions_.end()) {
            if (it->second->isExpired()) {
                LOG(INFO) << "Cleaning up expired session: " << it->first
                         << " (idle for " 
                         << it->second->getIdleDurationMinutes() 
                         << " minutes)";
                
                // Explicitly cleanup session resources before erasing
                it->second->cleanup();
                it = activeSessions_.erase(it);
            } else {
                ++it;
            }
        }
        
        LOG(INFO) << "Active sessions after cleanup: " << activeSessions_.size();
    }
    
    // Cleanup all sessions on shutdown
    void cleanupAllSessions() {
        std::lock_guard<std::mutex> lock(sessionsMutex_);
        
        LOG(INFO) << "Shutting down SessionManager, cleaning up " 
                  << activeSessions_.size() << " sessions";
        
        for (auto& [id, session] : activeSessions_) {
            session->cleanup();
        }
        
        activeSessions_.clear();
    }
    
    // Member variables
    mutable std::mutex sessionsMutex_;
    std::map<std::string, std::shared_ptr<ModelDevelopmentSession>> activeSessions_;
    std::chrono::minutes idleTimeout_;
    
    // Background cleanup
    std::atomic<bool> running_;
    std::thread cleanupThread_;
};

struct SessionStatistics {
    size_t totalSessions = 0;
    size_t activeSessions = 0;
    size_t expiredSessions = 0;
    size_t totalRequests = 0;
};
2. Model Development Session - Individual Session Management

//==============================================================================
// MODEL DEVELOPMENT SESSION - Per-User Session State
//==============================================================================

class ModelDevelopmentSession {
public:
    ModelDevelopmentSession(
        const std::string& sessionId,
        std::chrono::minutes idleTimeout
    ) : sessionId_(sessionId),
        idleTimeout_(idleTimeout),
        lastActivity_(std::chrono::steady_clock::now()),
        requestCount_(0),
        createdAt_(std::chrono::system_clock::now()) {
        
        LOG(INFO) << "Session " << sessionId_ << " created";
    }
    
    ~ModelDevelopmentSession() {
        LOG(INFO) << "Session " << sessionId_ << " destroyed. "
                  << "Processed " << requestCount_ << " requests";
    }
    
    // Process a request with delta optimization
    UniversalResult processRequest(
        const ValuationRequest& request,
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer
    ) {
        updateActivity();  // Update timestamp
        requestCount_++;
        
        LOG(INFO) << "Session " << sessionId_ 
                  << " processing request #" << requestCount_;
        
        // Detect what changed since last request
        RequestChanges changes = deltaTracker_.detectChanges(
            request,
            lastRequest_
        );
        
        UniversalResult result;
        
        if (changes.requiresFullRun()) {
            LOG(INFO) << "Full execution required: " << changes.getReason();
            result = executeFullRun(request, modelSuiteLayer);
        } else {
            LOG(INFO) << "Delta execution possible: " 
                      << changes.getChangedParameters().size() 
                      << " parameters changed";
            result = executeDeltaRun(request, changes, modelSuiteLayer);
        }
        
        // Store request for next delta comparison
        lastRequest_ = request;
        lastResult_ = std::make_shared<UniversalResult>(result);
        
        return result;
    }
    
    // Check if session has expired (idle too long)
    bool isExpired() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::minutes>(
            now - lastActivity_
        );
        return elapsed >= idleTimeout_;
    }
    
    // Get idle duration in minutes
    long getIdleDurationMinutes() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::minutes>(
            now - lastActivity_
        );
        return elapsed.count();
    }
    
    // Get session age
    long getAgeMinutes() const {
        auto now = std::chrono::system_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::minutes>(
            now - createdAt_
        );
        return elapsed.count();
    }
    
    // Get request count
    size_t getRequestCount() const {
        return requestCount_;
    }
    
    // Explicit cleanup of cached resources
    void cleanup() {
        LOG(INFO) << "Cleaning up session " << sessionId_;
        
        // Clear cached objects
        cachedPortfolio_.reset();
        loadedModels_.clear();
        baselineResults_.reset();
        lastResult_.reset();
        
        // Clear request history
        requestHistory_.clear();
        
        LOG(INFO) << "Session " << sessionId_ << " cleanup complete";
    }
    
    // Get session info for debugging/monitoring
    SessionInfo getInfo() const {
        SessionInfo info;
        info.sessionId = sessionId_;
        info.requestCount = requestCount_;
        info.idleMinutes = getIdleDurationMinutes();
        info.ageMinutes = getAgeMinutes();
        info.cacheSize = getCacheSize();
        info.isExpired = isExpired();
        return info;
    }
    
private:
    // Update last activity timestamp (called on every request)
    void updateActivity() {
        lastActivity_ = std::chrono::steady_clock::now();
    }
    
    // Execute full workflow (cold start or major change)
    UniversalResult executeFullRun(
        const ValuationRequest& request,
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer
    ) {
        // 1. Load or reload portfolio
        if (cachedPortfolio_ == nullptr || 
            request.portfolioSource != lastRequest_.portfolioSource) {
            
            LOG(INFO) << "Loading portfolio: " << request.portfolioSource;
            cachedPortfolio_ = modelSuiteLayer->loadPortfolio(
                request.portfolioSource
            );
        }
        
        // 2. Load models (cache by version)
        ensureModelsLoaded(request, modelSuiteLayer);
        
        // 3. Execute full workflow
        auto result = modelSuiteLayer->execute(request);
        
        // 4. Store as baseline for future delta runs
        baselineResults_ = std::make_shared<UniversalResult>(result);
        
        return result;
    }
    
    // Execute delta optimization (only recalculate what changed)
    UniversalResult executeDeltaRun(
        const ValuationRequest& request,
        const RequestChanges& changes,
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer
    ) {
        // Start with baseline results
        UniversalResult result = *baselineResults_;
        
        // Identify which suites need re-execution based on changed parameters
        auto affectedSuites = deltaTracker_.getAffectedSuites(
            changes,
            request.suites
        );
        
        LOG(INFO) << "Re-executing " << affectedSuites.size() 
                  << " affected suites";
        
        // Create modified request with only affected suites
        ValuationRequest deltaRequest = request;
        deltaRequest.suites = affectedSuites;
        
        // Execute only affected suites
        auto deltaResult = modelSuiteLayer->execute(deltaRequest);
        
        // Merge delta results into baseline
        result = mergeDeltaResults(result, deltaResult, affectedSuites);
        
        return result;
    }
    
    // Ensure required models are loaded and cached
    void ensureModelsLoaded(
        const ValuationRequest& request,
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer
    ) {
        for (const auto& suiteName : request.suites) {
            auto suiteKey = makeSuiteKey(suiteName, request);
            
            if (loadedModels_.find(suiteKey) == loadedModels_.end()) {
                LOG(INFO) << "Loading models for suite: " << suiteName;
                
                // Load and cache model suite
                auto models = modelSuiteLayer->loadModelsForSuite(
                    suiteName,
                    request.parameters
                );
                
                loadedModels_[suiteKey] = models;
            }
        }
    }
    
    // Merge delta results into baseline results
    UniversalResult mergeDeltaResults(
        const UniversalResult& baseline,
        const UniversalResult& delta,
        const std::vector<std::string>& affectedSuites
    ) {
        UniversalResult merged = baseline;
        
        // Replace results for affected suites
        for (const auto& suiteName : affectedSuites) {
            if (suiteName == "behavioral") {
                merged.behavioralResult = delta.behavioralResult;
            } else if (suiteName == "pricing") {
                merged.pricingResult = delta.pricingResult;
            } else if (suiteName == "cashflow") {
                merged.cashFlowResult = delta.cashFlowResult;
            }
        }
        
        return merged;
    }
    
    size_t getCacheSize() const {
        size_t size = 0;
        if (cachedPortfolio_) size += sizeof(*cachedPortfolio_);
        size += loadedModels_.size() * sizeof(void*);  // Rough estimate
        return size;
    }
    
    std::string makeSuiteKey(
        const std::string& suite,
        const ValuationRequest& request
    ) const {
        // Key includes suite name and relevant parameters
        return suite + "_" + request.parameters.at(suite).version;
    }
    
    // Member variables
    std::string sessionId_;
    std::chrono::minutes idleTimeout_;
    std::chrono::steady_clock::time_point lastActivity_;  // For idle detection
    std::chrono::system_clock::time_point createdAt_;     // For session age
    std::atomic<size_t> requestCount_;
    
    // Cached objects (lifetime = session)
    std::shared_ptr<Portfolio> cachedPortfolio_;
    std::map<std::string, std::shared_ptr<Model>> loadedModels_;
    std::shared_ptr<UniversalResult> baselineResults_;
    std::shared_ptr<UniversalResult> lastResult_;
    
    // Request tracking
    ValuationRequest lastRequest_;
    std::vector<ValuationRequest> requestHistory_;
    
    // Delta optimization
    DeltaChangeTracker deltaTracker_;
};

struct SessionInfo {
    std::string sessionId;
    size_t requestCount;
    long idleMinutes;
    long ageMinutes;
    size_t cacheSize;
    bool isExpired;
};
3. Delta Change Tracker - Detect What Changed
//==============================================================================
// DELTA CHANGE TRACKER - Detects Changes Between Requests
//==============================================================================

class DeltaChangeTracker {
public:
    // Detect what changed between current and previous request
    RequestChanges detectChanges(
        const ValuationRequest& current,
        const ValuationRequest& previous
    ) const {
        RequestChanges changes;
        
        // Check if this is the first request
        if (previous.portfolioSource.empty()) {
            changes.setFullRunRequired("First request in session");
            return changes;
        }
        
        // 1. Portfolio changed?
        if (current.portfolioSource != previous.portfolioSource) {
            changes.setFullRunRequired("Portfolio source changed");
            return changes;
        }
        
        // 2. Effective date changed?
        if (current.effectiveDate != previous.effectiveDate) {
            changes.setFullRunRequired("Effective date changed");
            return changes;
        }
        
        // 3. Suite list changed?
        if (current.suites != previous.suites) {
            changes.setSuitesChanged(
                findAddedSuites(current.suites, previous.suites),
                findRemovedSuites(current.suites, previous.suites)
            );
        }
        
        // 4. Parameters changed?
        detectParameterChanges(current, previous, changes);
        
        return changes;
    }
    
    // Determine which suites are affected by parameter changes
    std::vector<std::string> getAffectedSuites(
        const RequestChanges& changes,
        const std::vector<std::string>& requestedSuites
    ) const {
        std::vector<std::string> affected;
        
        // If parameters changed, determine which suites use those parameters
        for (const auto& [paramName, paramValue] : changes.getChangedParameters()) {
            // Look up which suites depend on this parameter
            auto suitesForParam = parameterSuiteMapping_.getAffectedSuites(paramName);
            
            for (const auto& suite : suitesForParam) {
                // Add if requested and not already in list
                if (std::find(requestedSuites.begin(), requestedSuites.end(), suite) 
                    != requestedSuites.end() &&
                    std::find(affected.begin(), affected.end(), suite) == affected.end()) {
                    affected.push_back(suite);
                }
            }
        }
        
        // Add downstream dependent suites
        affected = addDependentSuites(affected, requestedSuites);
        
        return affected;
    }
    
private:
    void detectParameterChanges(
        const ValuationRequest& current,
        const ValuationRequest& previous,
        RequestChanges& changes
    ) const {
        // Compare parameter maps
        for (const auto& [key, currentParam] : current.parameters) {
            auto it = previous.parameters.find(key);
            
            if (it == previous.parameters.end()) {
                // New parameter
                changes.addChangedParameter(key, currentParam);
            } else if (currentParam != it->second) {
                // Parameter value changed
                changes.addChangedParameter(key, currentParam);
            }
        }
        
        // Check for removed parameters
        for (const auto& [key, prevParam] : previous.parameters) {
            if (current.parameters.find(key) == current.parameters.end()) {
                changes.addRemovedParameter(key);
            }
        }
    }
    
    std::vector<std::string> findAddedSuites(
        const std::vector<std::string>& current,
        const std::vector<std::string>& previous
    ) const {
        std::vector<std::string> added;
        for (const auto& suite : current) {
            if (std::find(previous.begin(), previous.end(), suite) == previous.end()) {
                added.push_back(suite);
            }
        }
        return added;
    }
    
    std::vector<std::string> findRemovedSuites(
        const std::vector<std::string>& current,
        const std::vector<std::string>& previous
    ) const {
        std::vector<std::string> removed;
        for (const auto& suite : previous) {
            if (std::find(current.begin(), current.end(), suite) == current.end()) {
                removed.push_back(suite);
            }
        }
        return removed;
    }
    
    std::vector<std::string> addDependentSuites(
        const std::vector<std::string>& affected,
        const std::vector<std::string>& requested
    ) const {
        // If behavioral suite affected, pricing and cashflow also affected
        // (since they depend on behavioral)
        std::vector<std::string> result = affected;
        
        if (std::find(affected.begin(), affected.end(), "behavioral") != affected.end()) {
            if (std::find(requested.begin(), requested.end(), "pricing") != requested.end() &&
                std::find(result.begin(), result.end(), "pricing") == result.end()) {
                result.push_back("pricing");
            }
            if (std::find(requested.begin(), requested.end(), "cashflow") != requested.end() &&
                std::find(result.begin(), result.end(), "cashflow") == result.end()) {
                result.push_back("cashflow");
            }
        }
        
        return result;
    }
    
    ParameterSuiteMapping parameterSuiteMapping_;
};

//==============================================================================
// REQUEST CHANGES - What Changed Between Requests
//==============================================================================

class RequestChanges {
public:
    bool requiresFullRun() const { return fullRunRequired_; }
    std::string getReason() const { return reason_; }
    
    void setFullRunRequired(const std::string& reason) {
        fullRunRequired_ = true;
        reason_ = reason;
    }
    
    void setSuitesChanged(
        const std::vector<std::string>& added,
        const std::vector<std::string>& removed
    ) {
        addedSuites_ = added;
        removedSuites_ = removed;
    }
    
    void addChangedParameter(const std::string& name, const Parameter& value) {
        changedParameters_[name] = value;
    }
    
    void addRemovedParameter(const std::string& name) {
        removedParameters_.push_back(name);
    }
    
    const std::map<std::string, Parameter>& getChangedParameters() const {
        return changedParameters_;
    }
    
private:
    bool fullRunRequired_ = false;
    std::string reason_;
    std::vector<std::string> addedSuites_;
    std::vector<std::string> removedSuites_;
    std::map<std::string, Parameter> changedParameters_;
    std::vector<std::string> removedParameters_;
};

4. Interactive Application - Main Entry Point

//==============================================================================
// INTERACTIVE APPLICATION - Main Process
//==============================================================================

class InteractiveApplication {
public:
    InteractiveApplication(
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer,
        std::chrono::minutes sessionTimeout = std::chrono::minutes(30)
    ) : modelSuiteLayer_(modelSuiteLayer),
        sessionManager_(sessionTimeout),
        running_(false) {}
    
    // Start the interactive application
    void run() {
        LOG(INFO) << "Starting Interactive Application";
        
        // Start session manager (background cleanup)
        sessionManager_.start();
        running_ = true;
        
        // Start monitoring thread (for statistics/health checks)
        monitoringThread_ = std::thread(&InteractiveApplication::monitoringLoop, this);
        
        // Main request processing loop
        PresentationLayer presentation;
        presentation.initialize();
        
        try {
            while (running_) {
                // Get request from user (CLI/config/API)
                auto dto = presentation.getNextRequest();
                
                // Check for shutdown command
                if (dto.isShutdownCommand()) {
                    LOG(INFO) << "Shutdown command received";
                    break;
                }
                
                // Extract or generate session ID
                std::string sessionId = dto.arguments.count("session-id") > 0
                    ? dto.arguments["session-id"]
                    : generateSessionId();
                
                // Get or create session
                auto session = sessionManager_.getOrCreateSession(sessionId);
                
                // Build request
                ApplicationLayer app(modelSuiteLayer_);
                auto request = app.buildRequest(dto);
                
                // Process request through session (with delta optimization)
                auto result = session->processRequest(request, modelSuiteLayer_);
                
                // Display results
                presentation.displayResults(result);
                
                // Show session info
                auto sessionInfo = session->getInfo();
                LOG(INFO) << "Session " << sessionInfo.sessionId 
                         << " - Request #" << sessionInfo.requestCount
                         << " - Idle: " << sessionInfo.idleMinutes << "min";
            }
        } catch (const std::exception& e) {
            LOG(ERROR) << "Error in main loop: " << e.what();
        }
        
        shutdown();
    }
    
    // Graceful shutdown
    void shutdown() {
        LOG(INFO) << "Shutting down Interactive Application";
        
        running_ = false;
        
        // Stop monitoring thread
        if (monitoringThread_.joinable()) {
            monitoringThread_.join();
        }
        
        // Print final statistics
        auto stats = sessionManager_.getStatistics();
        LOG(INFO) << "Final Statistics:";
        LOG(INFO) << "  Total Sessions: " << stats.totalSessions;
        LOG(INFO) << "  Active Sessions: " << stats.activeSessions;
        LOG(INFO) << "  Total Requests: " << stats.totalRequests;
        
        // Stop session manager (cleanup all sessions)
        sessionManager_.stop();
        
        LOG(INFO) << "Shutdown complete";
    }
    
    // Get current statistics (for monitoring/debugging)
    SessionStatistics getStatistics() const {
        return sessionManager_.getStatistics();
    }
    
private:
    // Background monitoring loop (optional - for health checks, metrics, etc.)
    void monitoringLoop() {
        while (running_) {
            std::this_thread::sleep_for(std::chrono::minutes(1));
            
            if (!running_) break;
            
            // Log statistics periodically
            auto stats = sessionManager_.getStatistics();
            LOG(INFO) << "[Monitor] Active: " << stats.activeSessions
                     << ", Expired: " << stats.expiredSessions
                     << ", Total Requests: " << stats.totalRequests;
        }
    }
    
    std::string generateSessionId() const {
        // Generate unique session ID (e.g., timestamp + random)
        auto now = std::chrono::system_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            now.time_since_epoch()
        ).count();
        
        return "session_" + std::to_string(ms);
    }
    
    std::shared_ptr<ModelSuiteLayer> modelSuiteLayer_;
    SessionManager sessionManager_;
    std::atomic<bool> running_;
    std::thread monitoringThread_;
};

5. Lifecycle & Timing Diagram

Time: 0min                          Session Created
      ↓
      ├─ Request #1 (Full Run)      lastActivity = 0min
      │  - Load portfolio
      │  - Load models
      │  - Execute full workflow
      │  - Cache results
      │
Time: 2min
      ├─ Request #2 (Delta Run)     lastActivity = 2min
      │  - Reuse cached portfolio
      │  - Reuse loaded models
      │  - Only changed parameters
      │  - 10x faster!
      │
Time: 5min
      ├─ Request #3 (Delta Run)     lastActivity = 5min
      │  - Delta optimization
      │
Time: 10min
      ├─ Request #4 (Delta Run)     lastActivity = 10min
      │
      │
      │ [Idle Period - No Requests]
      │
Time: 40min                         lastActivity still = 10min
      │                             Idle Duration = 40 - 10 = 30min
      │                             Idle Duration >= Timeout (30min)
      │                             ↓
      │                             Session marked as EXPIRED
      │
Time: 45min
      │                             Cleanup thread runs
      │                             ↓
      │                             Session detected as expired
      │                             ↓
      │                             session->cleanup() called
      │                             - Clear cached portfolio
      │                             - Unload models
      │                             - Free memory
      │                             ↓
      │                             Session removed from active map
      │
      └─ Session DESTROYED          Memory 
6. Thread Safety & Concurrency

//==============================================================================
// THREAD SAFETY CONSIDERATIONS
//==============================================================================

/*
 * Threading Model:
 * 
 * 1. Main Thread:
 *    - Receives user requests
 *    - Calls sessionManager.getOrCreateSession()
 *    - Calls session.processRequest()
 * 
 * 2. Cleanup Thread (in SessionManager):
 *    - Periodically scans for expired sessions
 *    - Removes expired sessions
 *    - Mutex protects activeSessions_ map
 * 
 * 3. Monitoring Thread (optional):
 *    - Logs statistics
 *    - Health checks
 *    - Read-only access
 * 
 * Synchronization:
 * - SessionManager::activeSessions_ protected by mutex
 * - Each ModelDevelopmentSession is NOT thread-safe
 *   (assumes single-threaded access per session)
 * - If multi-threaded access needed per session, add session-level mutex
 */

class ThreadSafeModelDevelopmentSession {
public:
    UniversalResult processRequest(
        const ValuationRequest& request,
        std::shared_ptr<ModelSuiteLayer> modelSuiteLayer
    ) {
        // Protect entire request processing
        std::lock_guard<std::mutex> lock(sessionMutex_);
        
        // Same logic as before, now thread-safe
        // ...
    }
    
    bool isExpired() const {
        // lastActivity_ is atomic or protected
        std::lock_guard<std::mutex> lock(sessionMutex_);
        // ...
    }
    
private:
    mutable std::mutex sessionMutex_;  // Protect session state
};

7. Configuration & Tuning

//==============================================================================
// CONFIGURATION OPTIONS
//==============================================================================

struct SessionConfiguration {
    // Idle timeout before session expires
    std::chrono::minutes idleTimeout = std::chrono::minutes(30);
    
    // How often cleanup thread runs
    std::chrono::minutes cleanupInterval = std::chrono::minutes(5);
    
    // Maximum number of concurrent sessions
    size_t maxConcurrentSessions = 100;
    
    // Maximum session age (even if active)
    std::chrono::hours maxSessionAge = std::chrono::hours(24);
    
    // Enable request history tracking
    bool trackRequestHistory = true;
    size_t maxRequestHistory = 50;  // Keep last N requests
    
    // Cache settings
    size_t maxCacheSize = 10 * 1024 * 1024 * 1024;  // 10GB
    bool enableDeltaOptimization = true;
};

// Usage
InteractiveApplication app(modelSuiteLayer);
app.setConfiguration(config);



