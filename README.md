# TrainFlow — Workout Planner

TrainFlow is a personal training planning application built as a single-file React/Tailwind prototype. It connects to [intervals.icu](https://intervals.icu) to manage workouts, training phases, and fitness tracking.

## 🚀 Getting Started

To run the application locally, you need to serve it via HTTP to avoid CORS issues with the intervals.icu API.

```bash
# Serve the directory
python -m http.server 8080
```

Then open: [http://localhost:8080/trainflow.html](http://localhost:8080/trainflow.html)

## 🛠 Tech Stack

- **UI:** React 18 (CDN)
- **Styling:** Tailwind CSS (CDN)
- **Transpilation:** Babel Standalone (In-browser)
- **Data:** intervals.icu REST API
- **Persistence:** LocalStorage

## 📂 Project Structure

- `trainflow.html`: The main single-file application.
- `TRAINFLOW_HANDOFF.md`: Detailed handoff documentation and technical specs.
- `.agent/`: Automation and agent-specific configurations/workflows.

## 📝 Best Practices

- **Version Control:** Use Git for all changes.
- **Documentation:** Maintain `TRAINFLOW_HANDOFF.md` with architectural updates.
- **Safety:** Avoid committing sensitive API keys if possible (currently hardcoded in `trainflow.html`).
