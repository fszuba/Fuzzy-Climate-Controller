# Fuzzy Climate Controller

A JavaFX application that simulates humidity control using Mamdani fuzzy inference. The controller takes temperature and relative humidity as inputs and calculates how strongly to humidify or dehumidify a room.

The simulation is intended for experimenting with fuzzy rules and membership functions. It is not a model of a real HVAC system.

<img width="1225" height="919" alt="dashboard" src="https://github.com/user-attachments/assets/510a9393-1d0d-4406-8422-52156e53e94b" />


## Requirements

- JDK 21 or newer
- Maven

Run the application:

```bash
mvn clean javafx:run
```

Build the project:

```bash
mvn clean package
```

Run the tests:

```bash
mvn test
```

## How it works

The controller produces a value between `-1.0` and `1.0`:

- Negative values request dehumidification.
- `0.0` means no adjustment.
- Positive values request humidification.

The magnitude determines the strength of the correction. Instead of choosing a single matching rule, the engine evaluates all active rules and combines their output sets before calculating the final signal.

### Input variables

Temperature is defined over the range `20-27 °C`.

| Category | Membership function | Parameters |
| --- | --- | --- |
| `VERY_COLD` | Left shoulder | 1 at 20.00; 0 at 21.75 |
| `COLD` | Triangle | 20.00, 21.75, 23.50 |
| `COMFORTABLE` | Triangle | 21.75, 23.50, 25.25 |
| `WARM` | Triangle | 23.50, 25.25, 27.00 |
| `VERY_HOT` | Right shoulder | 0 at 25.25; 1 at 27.00 |

Relative humidity is defined over the range `0-100%`.

| Category | Membership function | Parameters |
| --- | --- | --- |
| `DRY` | Left shoulder | 1 at 30; 0 at 50 |
| `COMFORTABLE` | Trapezoid | 35, 45, 55, 65 |
| `HUMID` | Right shoulder | 0 at 50; 1 at 70 |

The output uses five fuzzy sets over `[-1, 1]`.

| Action | Center or endpoint | Membership function | Parameters |
| --- | ---: | --- | --- |
| `STRONG_DEHUMIDIFICATION` | -1.0 | Left shoulder | 1 at -1.0; 0 at -0.5 |
| `DEHUMIDIFICATION` | -0.5 | Triangle | -1.0, -0.5, 0.0 |
| `MAINTAIN` | 0.0 | Triangle | -0.5, 0.0, 0.5 |
| `HUMIDIFICATION` | 0.5 | Triangle | 0.0, 0.5, 1.0 |
| `STRONG_HUMIDIFICATION` | 1.0 | Right shoulder | 0 at 0.5; 1 at 1.0 |

The membership functions are triangular, trapezoidal, or shoulder-shaped. Each returns a degree in `[0, 1]`, with overlapping sets around category boundaries. Their parameters are validated when the functions are created.

### Rule base

The rule base contains 15 rules, covering every temperature and humidity category combination.

| Temperature / Humidity | `DRY` | `COMFORTABLE` | `HUMID` |
| --- | --- | --- | --- |
| `VERY_COLD` | `STRONG_HUMIDIFICATION` | `HUMIDIFICATION` | `MAINTAIN` |
| `COLD` | `STRONG_HUMIDIFICATION` | `HUMIDIFICATION` | `MAINTAIN` |
| `COMFORTABLE` | `HUMIDIFICATION` | `MAINTAIN` | `DEHUMIDIFICATION` |
| `WARM` | `MAINTAIN` | `DEHUMIDIFICATION` | `STRONG_DEHUMIDIFICATION` |
| `VERY_HOT` | `MAINTAIN` | `DEHUMIDIFICATION` | `STRONG_DEHUMIDIFICATION` |

Humidity determines the starting action. Dry conditions favor humidification, comfortable humidity favors no change, and high humidity favors dehumidification. Temperature shifts that action by one level: toward humidification when cold and toward dehumidification when warm. The result is limited to the five available actions.

This is a chosen rule policy for the simulator, not a calibrated recommendation for real indoor climate control. In particular, a very hot but dry room and a very cold but humid room both receive a neutral action rather than a correction that could make the humidity more extreme.

### Inference and defuzzification

The engine uses the Mamdani method:

1. Calculate membership degrees for both inputs.
2. Evaluate each rule using `min(temperatureMembership, humidityMembership)`, multiplied by the rule weight.
3. Clip each rule's output membership function to its activation level.
4. Combine the clipped output sets with `max`.
5. Calculate the centroid of the aggregated set to obtain the control signal.

The centroid is calculated as:

```text
controlSignal = integral(x * membership(x)) / integral(membership(x))
```

Both integrals cover `[-1, 1]` and are approximated using the composite trapezoidal rule with 1,000 intervals by default. If the aggregated set has negligible area, the engine returns `0.0`.

The inference result also contains the input membership degrees, active rules and their strengths. The JavaFX dashboard displays these values alongside the control signal.

## Simulation

Each simulation step updates relative humidity using the current control signal:

```text
humidityChange = maximumHumidityChangePercentPerStep * controlSignal
```

By default, the simulation runs one step every two seconds, with a maximum change of two percentage points per step. Changes smaller than the configured control deadband (5% by default) are ignored. Humidity is restricted to the range `0-100%`.

The simulation supports start, pause, resume and reset. The dashboard shows the current measurements, control action, rule activations and a humidity history chart.

The humidity model is deliberately simple. It does not account for room volume, ventilation, condensation, evaporation rates or sensor noise. Temperature is an input to the controller, not a simulated thermodynamic variable.

## Code structure

The project separates JavaFX presentation code from the controller and simulation logic:

| Component | Responsibility |
| --- | --- |
| `HumidityControlApplication`, `DashboardController` | Application startup and JavaFX interface |
| `FuzzyInferenceEngine` | Rule evaluation and output aggregation |
| `FuzzyRuleBase`, `FuzzyRule` | Rule definitions |
| `MembershipFunction` and its implementations | Fuzzy set membership calculations |
| `CentroidDefuzzifier` | Conversion of an aggregated fuzzy set to a numeric value |
| `FuzzyInferenceResult` | Inference output and diagnostics |
| `SimulationEngine`, `SimulationConfiguration` | Simulation state and update parameters |
| `EnvironmentState` | Temperature and humidity values |

All Java classes use the `it.farrlor` package. The inference and simulation code do not depend on JavaFX and can be tested without launching the UI.

## Tests

The JUnit 5 tests cover membership function boundaries, rule selection and activation, aggregation, defuzzification, simulation state changes and input validation. Numerical assertions use tolerances where appropriate.

```bash
mvn test
```

### Example results

The table below shows representative results for the default configuration. Values are approximate.

| Temperature | Humidity | Control signal | Action |
| --- | --- | ---: | --- |
| 23.5 °C | 20% | +0.50 | Humidification |
| 23.5 °C | 50% | 0.00 | Maintain |
| 23.5 °C | 85% | -0.50 | Dehumidification |
| 20.0 °C | 10% | +0.83 | Strong humidification |
| 27.0 °C | 90% | -0.83 | Strong dehumidification |
| 27.0 °C | 15% | 0.00 | Maintain |
| 20.0 °C | 90% | 0.00 | Maintain |

## Possible extensions

- Load membership functions and rules from an external configuration file.
- Add another output variable, such as fan speed.
- Compare Mamdani inference with a Sugeno controller.
- Plot the output membership functions and the aggregated fuzzy set.
- Replace the current simulation model with one that accounts for temperature, ventilation and moisture exchange.
