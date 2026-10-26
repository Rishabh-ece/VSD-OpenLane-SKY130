# 08 — Power distribution and routing

[← Back to project README](../README.md)

## Objective and theory

The **PDN** distributes power and ground through stripes, rails, and vias. **Routing** connects signal nets over legal metal tracks; routing completion is distinct from final timing and physical verification.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
# After the preceding stages
run_routing
# Inspect routing logs, timing reports, and physical checks
```

## Screenshot-by-screenshot explanation

### Step 1 — Successful PDN generation with warnings

The power-grid tool reports successful PDN generation and connectivity, while also warning about moved via/stripe positions and power assumptions. These warnings should be reviewed rather than interpreted as sign-off.

![Successful PDN generation with warnings](../images/61_pdn_gen.png)

### Step 2 — Routing completed, timing not closed

The final screenshot reports routing completed, with **WNS −15.76 ns** and **TNS −339.89 ns** for that run. Routing completion confirms progress through the flow but does not mean setup timing, DRC, or LVS closure.

![Routing completed, timing not closed](../images/62_routing_success.png)

## Key takeaway

The flow reached routing completion, but negative WNS/TNS and warnings mean sign-off is not demonstrated.
