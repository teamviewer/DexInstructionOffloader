TV1 Instruction Offloader

Automated scheduling and export engine for 1E / Tachyon instructions.

TV1 Instruction Offloader executes instructions on a schedule, enforces TTL-based export boundaries, and exports results using:

    Pagination export - page by page parsing of the results 
    Server-side export - sets export = true when executing the instruction 
    Both (pagination + server)

It includes export queuing, retry handling, restart recovery, execution history tracking, and chart-based analytics.
📦 Requirements

        .NET 8
        Windows 11 (primary target) may work withj Mac and Linux as well
        Valid DEX platform access
        Export folder write permissions
        Non-Interactive authorization - cetificate based authentication 
                JWT Prinipal Mapping setup for a service account with execute and view access to the instructions that will be automated

✨ Core Features
  🔹 Scheduled Instruction Execution
    Per-job scheduling
    Platform Alias + FQDN support
    Certificate authentication or interactive authentication
    Multiple concurrent platforms supported
    Per-execution isolation using ResponseId + ExecutionId tracking
    
🔹 Export Modes

    Each job supports one of the following modes:
    Mode	Name	Description
        0	PagingOnly	Exports using paginated API calls only
        1	ServerOnly	Waits for server-side export and downloads it
        2	PagingThenServer	Runs pagination export first, then performs server-side export once available

        

⏳ TTL-Based Export Model

    Exports do not start immediately after instruction execution
    Exports begin after Instruction TTL expiration:
    Paging export runs after Instruction TTL ends
    Server-side export polling begins after Instruction TTL ends
    Response TTL is used only for expiration decisions (not export timing)
    This sohuld ensure:
        Complete datasets
        No partial exports
    Deterministic export windows
    No race conditions between instruction execution and export

🔁 Retry & Fallback Logic

        PagingOnly
                Retries only on API failure or restart recovery
                Does not retry because file was not immediately visible
                Optional fallback to server-side export after configurable failure threshold
                Once paging succeeds, export is marked complete and does not retry
        ServerOnly
                Polls until server-side export becomes available
                Retries polling on timeout
                Stops after max retry window (default: 3)
        PagingThenServer
                Paging export runs once after TTL
                Server-side export runs independently after TTL
                Paging success does not block server export
                Each phase is isolated and tracked separately

📊 Execution History

    History tracks per-execution (not per job):
        ExecutionId
        ResponseId
        Status
            (Exported, PagingExported, WaitingServerExport, NoData, Expired, etc.)
        Paging file path
        Server file path
        Server download URL
        Retry count
        Duration
        Platform alias / FQDN
        TTL values used
        History:
        Updates live
        Persists across restarts
        Deduplicates per execution
        Normalizes state after recovery

🔄 Recovery on Restart

    On startup:
        In-flight exports are resumed
        Pending server exports are re-polled
        Paging exports are retried if incomplete
        Stale execution states are normalized
        Recovery should not:
            Re-run successful exports
            Duplicate completed paging exports
            Reset completed execution state

🗂 Export Output Structure

    Generated files:
        {InstructionName}_{ResponseId}_P_yyyyMMdd_HHmmss.ext
        {InstructionName}_{ResponseId}_S_yyyyMMdd_HHmmss.tsv              
    
        Folder modes- Configured per platform per job via Export Options.:
                By Platform
                By Instruction
                By Date
                Instruction → Date
        

📈 History Charts

      The History tab includes visual analytics:
      Jobs per Day of Week
      Jobs per Hour
      Duration Buckets
      Heatmap / Concurrent Load
      Average Duration
      Charts are based on ExecutionHistory, not schedule definitions.

🕒 Local vs UTC

    Toggle affects:
        History grid timestamps
        Chart grouping boundaries (hour/day bucketing)
        Charts recompute when toggled
        Duration values are timezone-neutral
            Note: Some visual adjustments may still be refined withn the charts.

⚠️ Current Known Issues

        1️⃣ Paging File Visibility Timing (Rare / Slow Disk)
                On slower systems, the paging file may not be immediately visible after completion.
                Current behavior:
                Paging success is determined by API completion, not file existence.
                Transient "WaitingExport" may appear briefly before finalization.
        
        2️⃣ History Status Drift After Restart (Rare)
        
            Under specific restart timing:
            A previously completed export may briefly show WaitingExport.
            Status normalizes on next scheduler tick.
            Does not affect export correctness.
        
        3️⃣ UI Layout Scaling on Some Windows 10 VMs
        
            Certain DPI combinations may clip top tab headers.
            Manual scaling logic was removed to stabilize layout.
            Some environments may still require layout refinement.
        
        4️⃣ Zero-Row Detection (Paging)
        
            Currently:
                Paging success = API call completed
                Row count is not directly inspected
                Future enhancement will:
                Detect zero-row result sets deterministically
                Mark NoData explicitly from API result

🚀 Planned Enhancements

🔹 1. True Row Count Detection

        Detect zero results directly from paging API if it is available
        Mark NoData deterministically
        Avoid ambiguity from file existence checks

🔹 2. Per-Platform Execution Isolation

        Dedicated execution threads per platform
        Prevent token context switching issues
        Reduce concurrent authentication contention

🔹 3. Persistent Export Queue Store

        Save pending export queue to disk
        Resume export position across restarts
        Support large multi-page recoveries

🔹 4. Export Health Dashboard

        Retry count visualization
        Per-job export metrics
        Failure analytics

🔹 5. Configurable Retry Thresholds

        AppSettings support for:
                Paging fallback threshold
                Server polling interval
                Max retry window

🔹 6. Headless / Tray-Only Mode

        Optional background-only mode
        System tray controls
        Run at startup
        No full UI required
        Export orchestration logic lives inside:
        JobSchedulerService.QueueExport()
        Maybe create a service but not everyone will have admin access to do this so i left ti as a startup option 
    
