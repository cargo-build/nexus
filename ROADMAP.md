- [ ] Clean up code (e.g., make fields private in non-data structs)
- [ ] Add Vim keybindings to the TUI
- [ ] Load quantized models directly without RAM preprocessing without llama cpp
- [ ] Revisit service decomposition only against a measured problem: the
  Python service and the Rust orchestrator are already separate processes over
  gRPC. Acceptance: a documented target for deployment friction, latency,
  failure isolation, or throughput that the current boundary cannot meet —
  measured before and after — plus a comparison of in-process modularization
  versus a message broker. Adopt a broker only with that evidence.
- [ ] Rebuild the Redis key structure from flat to complex
- [ ] Implement hybrid search
- [ ] Implement RLM
- [ ] Handle parsing errors via a separate sub-agent
