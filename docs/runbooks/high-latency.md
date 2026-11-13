# Runbook: HighLatencyP99

**Severity:** warning (posted to #alerts-warning)
**Fires when:** the slowest 1% of a service's requests take over 1 second for 5 minutes, in staging or production.

## What it means
Most customers are fine, but a noticeable share wait too long. Left alone, slowness often turns into errors.

## Check
1. Open **Service Overview (RED)** with **Environment** set to the alert's namespace. Which service has the high p99? Is the error rate also rising?
2. Find slow traces in Jaeger: service `gateway`, **Min Duration** `1s`, then **Find Traces**. The longest bar in the waterfall is the slow step.
3. Open **Infrastructure (Node)**. Is CPU or memory near its limit?

## Common causes and fixes
- **A slow or missing service further down the chain.** The gateway waits up to 5 seconds for a reply, the order service up to 3 seconds. In the earlier observability project (November 11, 2026), gateway p99 rose to about 2.3 seconds while product was down. Fix the service the trace points to.
- **The node is overloaded.** Also seen in the earlier project as p99 spikes while the laptop was busy. Reduce load, or raise CPU limits in `base/<service>/deployment.yaml` through Git if pods are throttled (check the built-in CPUThrottlingHigh alert).
- **Cache unavailable**, so every request goes to Postgres. Check `kubectl logs deploy/product -n <namespace> | grep -i redis`.

## Confirm it is fixed
The p99 line falls back under 1 second, and Slack posts **[RESOLVED] HighLatencyP99**.

## Escalate
If latency keeps rising or errors appear, treat it as HighErrorRate.
