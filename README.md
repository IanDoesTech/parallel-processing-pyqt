# Parallel Processing From PyQt6

This repository is a small PyQt6 demo that keeps the GUI responsive while work runs outside the GUI thread.

It combines a Qt `QThread` with Ray actors. The Qt thread acts as the bridge between the user interface and the Ray process, and the Ray actor produces a steady stream of results that are sent back to the GUI with Qt signals.

## What This Shows

- Keep long-running work off the PyQt GUI thread.
- Use a `QThread` worker as the boundary between Qt and parallel processing.
- Send control messages and results through Ray queues.
- Update the GUI from Qt signals instead of touching widgets from worker code.
- Keep the example organized with a lightweight Model-View-ViewModel structure.

## Requirements

- Python 3.11+
- PyQt6
- Ray

This is a desktop GUI example. It needs an environment where Qt can open a window.

## Setup

From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirement.txt
```

## Run The Demo

```bash
python3 main.py
```

Click **Start** to begin processing. The label updates once per second with the latest value produced by the Ray actor. Click **Stop** before closing the window.

The process also prints generated values to the console, which makes it easier to see that the worker path is running separately from the GUI event loop.

## Project Layout

```text
main.py                         # Starts the QApplication and main window
view/
  view.py                       # QMainWindow and widget updates
viewmodel/
  viewmodel.py                  # Button handling and view-facing signals
model/
  model.py                      # Owns the QThread
  thread_worker.py              # Bridges Qt signals and Ray queues
parallel/
  parallel_supervisor.py        # Ray actor supervising background work
  data_generator.py             # Ray actor producing demo data
helpers/
  process_instructions.py       # Queue message types for process control
  view_instructions.py          # Signal payload types for view updates
```

## How The Flow Works

The GUI never calls the Ray actor directly.

When the start button is clicked, the `ViewModel` tells the `Model` to start a `QThread`. The `ThreadWorker` has already been moved onto that thread, so its `start()` method runs away from the GUI event loop.

The `ThreadWorker` then creates a Ray `ParallelSupervisor` actor and passes it two queues:

```text
instruction_queue -> messages from Qt to Ray
result_queue      -> generated values from Ray back to Qt
```

`ParallelSupervisor` repeatedly asks `DataGenerator` for the next integer and puts the result on `result_queue`. The Qt worker reads that queue and emits a Qt signal. The view receives the signal and updates the label on the GUI side.

Stopping sends a `QuitProcessing` instruction through the instruction queue, then quits and waits for the Qt thread.

## Adapting The Example

For simple GUI applications, a `QThread` may be enough. The useful Ray-specific part of this example is the boundary between Qt and work that must happen in a separate process.

In a real application, `DataGenerator` would usually become a worker that does CPU-heavy processing, talks to a streaming API, or calls into a separate service. The `QuitProcessing` message also gives you a place to add more process-control instructions later, such as pausing, resuming, changing subscriptions, or flushing state.

For a related example that simulates streaming stock-price data and processing per-symbol results in Ray actors, see [ray-streamer](https://github.com/IanAtDazed/ray-streamer).

## Notes

This is teaching code, not a production framework. Production GUI applications usually need broader lifecycle handling, error propagation, cancellation, logging, tests around start/stop behavior, and a cleaner shutdown path for all Ray actors.

The important pattern to copy is:

```text
GUI thread -> QThread worker -> Ray queues and actors -> Qt signal -> GUI update
```

## Reference Sources

- [Use PyQt's QThread to Prevent Freezing GUIs](https://realpython.com/python-pyqt-qthread/)
- [Doing Python multiprocessing the right way](https://medium.com/@sampsa.riikonen/doing-python-multiprocessing-the-right-way-a54c1880e300)
- [A Clean Architecture for a PyQt GUI Using the MVVM Pattern](https://medium.com/@mark_huber/a-clean-architecture-for-a-pyqt-gui-using-the-mvvm-pattern-b8e5d9ae833d)
- [Model-View-ViewModel](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)
- [Ray Core actors](https://docs.ray.io/en/latest/ray-core/actors.html)
