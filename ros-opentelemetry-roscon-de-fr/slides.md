# ros-opentelemetry

<!-- npx reveal-md --host 0.0.0.0 --port 5252 --highlight-theme github --theme solarized slides.md -o ouput.html -->

End‑to‑End Telemetry for Robotics

---

### About me

- Lead Software & Robotics Engineer at Circu Li-ion
- Open-source Contributor

---

### ROS is a distributed system

"_ROS was invented at the moment when micro-services architecture was a hype._"

Guillaume Binet, Founder @ Copper Robotics

---

- ROS Node == MicroService
- DDS == Communication Layer

- How people handle it?

---

### Distributed Tracing

Traces give us the big picture of what happens when a request is made to an application.

Whether your application is a monolith with a single database or a sophisticated mesh of services, traces are essential to understanding the full “path” a request takes in your application.

_- opentelemetry.io_ 

---

!["production trace view"](./static/prod_traces.png)
*Diassembly request going though our system*

---
### Correlated logs

Correlating traces and logs allows you to investigate issues by navigating from a specific span in a trace directly to the logs that were generated during that operation.

This makes debugging faster and more intuitive by providing the exact context needed to understand an error or performance problem.

_- datadoghq.com_

---

!["selecting a trace"](./static/correlated_logs_trace.png)
*Selecting a trace marked with error*

---

!["selecting a trace"](./static/correlated_logs_log_view.png)
*Exploring correlated logs*

---

### Metrics

"A measurement captured at runtime."

_- opentelemetry.io_

---

!["metrics dashboard"](./static/metrics_dashboard.png)
*no more ssh/htop to remote machine*

---

### Why OpenTelemetry?

- An observability framework and toolkit designed to facilitate the **Generation**, **Export**, **Collection** of **traces**, **metrics**, and **logs**.
- Open source, Vendor- and tool-agnostic 
- Steered by Cloud Native Computing Foundation / Linux Foundation

---

*vendors supporting opentelemetry protocol*
!["metrics dashboard"](./static/otel_vendors.png)

---

### ROS2 x OpenTelemetry = ros-opentelemetry

!["ros-opentelemetry-github"](./static/ros-opentelemetry-github.png)

---

### Instrumenting your code

---

### Message Interface

```
# Goal
RobotTarget[] targets

ros_opentelemetry_interfaces/TraceMetadata trace_metadata # <-- trace interface
\---
# Result
bool success
bool[] targets_status
\---
# Feedback
uint32 current_target_id
bool status
```

```xml
  <depend>ros_opentelemetry_interfaces</depend>
```
---

### Setting up Tracer

```cpp
#include "ros_opentelemetry_cpp/ros_opentelemetry_cpp.hpp"

std::string otlp_grpc_endpoint = "hostname-of-your-otel-collector:4317";
ros_opentelemetry_cpp::setup_tracer("robot_control", otlp_grpc_endpoint);
```

```python

from ros_opentelemetry_py import setup_tracer

# somewhere at the start of your node
if __name__ == "__main__":
    # Expects environment variable OTLP_ENDPOINT set
    setup_tracer("robot_task_producer")

```
---
### Tracing Code

```cpp
#include <opentelemetry/trace/span.h>
#include <opentelemetry/trace/tracer.h>
#include <opentelemetry/trace/provider.h>
#include <opentelemetry/trace/scope.h>


auto tracer = opentelemetry::trace::Provider::GetTracerProvider()->GetTracer(
      "name_of_your_component");
auto span = tracer->StartSpan("handleActionOrServiceOrOtherCallback");
{

    auto target_span = tracer->StartSpan("nested_span");
    opentelemetry::trace::Scope scope(span);
    // your code
}
```

---

```python
from opentelemetry import trace

tracer: trace.Tracer = trace.get_tracer(__name__)

@tracer.start_as_current_span("method_of_your_node")
def method_of_your_node(self, params):
    ...
```

---

### Injecting Trace Context

```python
from ros_opentelemetry_py import inject_trace_context

example_msg = ExampleActionMessage.Goal()
example_msg.trace_metadata = inject_trace_context()
```

---

### Extracting Trace Context

```cpp
#include <opentelemetry/context/runtime_context.h>

const auto goal = goal_handle->get_goal();
auto extracted_ctx =
ros_opentelemetry_cpp::extract_trace_context(&goal->trace_metadata);
[[maybe_unused]] auto ctx_token =
      opentelemetry::context::RuntimeContext::Attach(extracted_ctx);
```

---

### Correlating Logs

```cpp
RCLCPP_ERROR_TRACED(this->get_logger(), "logger")
```

```python
from ros_opentelemetry_py import wrap_logger

self._traced_logger = wrap_logger(self.get_logger())
```

---

### Collecting & Exporting Logs

```yaml
receivers:
  filelog:
    include: ["/opt/logs/**/*.log"]
    start_at: end
    multiline:
      line_start_pattern: '^\[\w+\] \[\d+\.\d+\] \[.*\]:'  # To support cases when we output multiline json
    operators:
      - type: regex_parser
        regex: '^\[(?P<level>\w+)\] \[(?P<timestamp>\d+\.\d+)\] \[(?P<source>[^\]]+)\]: (?P<message>.*)$'
        timestamp:
          parse_from: attributes.timestamp
          layout_type: epoch
          layout: "s.ns"
        severity:
          parse_from: attributes.level
      - type: regex_parser
        parse_from: attributes.message
        regex: '^(?:\[trace_id=(?P<trace_id>[0-9a-f]{32})\s+span_id=(?P<span_id>[0-9a-f]{16})\]\s*)?(?P<body>.*)$'
...
```

---

### Demo Example

```
# To run example telemetry setup
just docker-up-telemetry

# Run example
just docker-up-example
```
---

## Thanks for listening

- [linkedin/szobovdev](linkedin.com/in/szobovdev/)
- [github/szobov](github.com/szobov)
- [blog.szobov.ru](blog.szobov.ru/)
