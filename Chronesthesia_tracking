from openexp.canvas import canvas       # drawing surface (text/lines) for the task
from openexp.mouse import mouse         # mouse input (click + position polling)
from openexp.keyboard import keyboard   # keyboard polling (ESC abort)
from psychopy import core               # clock timing + short waits (core.wait)
import os                               # Filesystem utilities (paths, folders)
import pandas as pd                     # read tracking CSV to plot trajectories
import matplotlib.pyplot as plt         # create/save trajectory plot
from datetime import datetime           # timestamped filenames


def format_time(milliseconds):          # Utility: ms -> seconds string with 3 decimals
    seconds = milliseconds / 1000
    return f"{seconds:.3f}"


rt_clock = core.Clock()                  # Global clock: timestamps drag onset/samples/placement timing
experiment_finished = False              # Control flag: stop loops cleanly (finish or ESC)


try:
    my_canvas = canvas(exp)                                     # Task canvas used for all drawing
    my_mouse = mouse(exp, visible=True)                         # Mouse object; cursor forced visible
    my_keyboard = keyboard(exp, keylist=['escape'], timeout=0)  # Non-blocking ESC polling
    holding_time = 2000                                         # Placement criterion: must hold on target line for this long (ms)

    target_line_y = 0                                           # Y coordinate of target line
    line_threshold = 20                                         # Tolerance area around target line (± pixels)

    subject_nr = var.subject_nr                                 # Subject identifier taken from OpenSesame variable space

    letters = {                                                 # Per-letter state: position, placed flag, drag start time
        'N': {'text': 'N', 'x': 0, 'y': 300, 'placed': False, 'start_time': None},
        'P': {'text': 'P', 'x': 0, 'y': 300, 'placed': False, 'start_time': None},
        'F': {'text': 'F', 'x': 0, 'y': 300, 'placed': False, 'start_time': None}
    }

    tracking_records = []                                       # Buffer: continuous drag samples (time, x, y, letter)
    placement_records = []                                      # Buffer: one summary row per placed letter (duration, final coords)

    def redraw_canvas(current_letter=None):                     # Rendering helper: clears, draws line + letters, shows canvas, enforces cursor
        my_canvas.clear()
        my_canvas.line(-321, target_line_y, 321, target_line_y)
        for key, letter in letters.items():
            if key == current_letter or letter['placed']:
                color = '#00FF00' if letter['placed'] else 'white'
                my_canvas.text(letter['text'], center=True, x=letter['x'], y=letter['y'], color=color)
        my_canvas.show()
        my_mouse.show_cursor()

    selected_letter = None                                      # Drag state: currently grabbed letter (None if not dragging)
    last_update_time = rt_clock.getTime()                       # Throttle redraws to reduce CPU load / flicker during dragging

    for current_letter in ['N', 'P', 'F']:                      # Main loop: run the drag-to-line task for each letter
        letters[current_letter]['placed'] = False
        letters[current_letter]['x'] = 0
        letters[current_letter]['y'] = 300
        letters[current_letter]['start_time'] = None
        selected_letter = None
        start_holding_time = None                                # Holding timer: when the cursor first enters the target band

        my_mouse.flush()                                         # Clear pending mouse events to avoid stale clicks
        redraw_canvas(current_letter)                            # Initial draw for this letter

        while not letters[current_letter]['placed']:                # Per-letter interaction loop until placement criterion met
            button, pos, timestamp = my_mouse.get_click(timeout=0)  # Non-blocking polling of click + cursor position

            if button and selected_letter is None:                  # Chunk: "grab" letter if click occurs near its current position
                x, y = pos
                if (not letters[current_letter]['placed']
                    and abs(x - letters[current_letter]['x']) < 20
                    and abs(y - letters[current_letter]['y']) < 20):
                    selected_letter = current_letter
                    if letters[current_letter]['start_time'] is None:
                        letters[current_letter]['start_time'] = rt_clock.getTime()  # Start timing at first successful grab
                    redraw_canvas(current_letter)

            if button and selected_letter:                          # Chunk: dragging + logging + placement check while mouse held down
                x, y = pos
                current_time = rt_clock.getTime()

                if letters[selected_letter]['x'] != x or letters[selected_letter]['y'] != y:  # Log only on movement
                    letters[selected_letter]['x'], letters[selected_letter]['y'] = x, y

                    if current_time - last_update_time > 0.05:                                # Throttle redraw to ~20 Hz for stability/performance
                        redraw_canvas(current_letter)
                        last_update_time = current_time

                    relative_drag_time = (current_time - letters[current_letter]['start_time']) * 1000   # Time since drag start (ms)
                    formatted_time = format_time(relative_drag_time)
                    tracking_records.append((subject_nr, formatted_time, x, y, selected_letter))         # Store trajectory sample

                if abs(y - target_line_y) <= line_threshold and not start_holding_time:                  # Start hold timer when entering band
                    start_holding_time = current_time
                elif abs(y - target_line_y) > line_threshold and start_holding_time:                     # Reset hold timer if leaving band
                    start_holding_time = None

                if start_holding_time and (current_time - start_holding_time) * 1000 >= holding_time:    # Chunk: placement rule met
                    if letters[selected_letter]['x'] < -321:                                             # Clamp final x to line segment
                        letters[selected_letter]['x'] = -321
                    elif letters[selected_letter]['x'] > 321:
                        letters[selected_letter]['x'] = 321

                    letters[selected_letter]['placed'] = True                                             # Mark letter as placed (turns green)
                    var.position_x = letters[selected_letter]['x']                                        # Save final x into OpenSesame vars
                    var.letter = current_letter                                                           # Save placed letter
                    var.rt = format_time((current_time - start_holding_time) * 1000)                      # Save hold duration/RT in seconds string
                    log.write_vars()                                                                      # Write OpenSesame variables to the log

                    drag_end_time = current_time
                    duration = (drag_end_time - letters[selected_letter]['start_time']) * 1000            # Total drag duration (ms)

                    formatted_duration = format_time(duration)
                    tracking_records.append((subject_nr, formatted_duration, x, y, selected_letter))      # Store final trajectory timestamp
                    placement_records.append((subject_nr, current_letter, formatted_duration,
                                              f"{int(letters[selected_letter]['x'])}, {int(target_line_y)}"))  # Store placement summary

                    redraw_canvas(current_letter)                                                              # Final redraw with placed letter
                    break

            elif not button and selected_letter is not None:  # Chunk: mouse released -> drop (stop dragging) and reset holding state
                selected_letter = None
                start_holding_time = None

            key, time_key = my_keyboard.get_key()             # Chunk: keyboard abort check (runs in parallel with dragging)
            if key == 'escape':
                print("The experiment was stopped prematurely by the user.")
                experiment_finished = True
                break

        if experiment_finished:                               # Chunk: break out of outer loop if ESC triggered
            break

        redraw_canvas()                                       # Chunk: inter-letter refresh (show placed letters)
        core.wait(0.05)                                       # Small wait to reduce CPU usage and keep display stable

    experiment_finished = True                                # Chunk: normal completion (all letters processed)

except Exception as e:                                        # Chunk: error handling to avoid silent failures
    print(f"An error occurred: {e}")

if experiment_finished:                                       # Chunk: export + plotting + final screenshot
    print("All letters have been placed, ending the experiment.")

    path_to_save = os.path.expanduser('~/Desktop/Lorenzo/Risultati chronesthesia')  # Output directory
    os.makedirs(path_to_save, exist_ok=True)                                        # Ensure output directory exists

    current_time = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")                     # Timestamp for unique filenames

    file_name_placement = f"subject_{subject_nr}_placement_{current_time}.csv"      # Placement summary filename
    file_path_placement = os.path.join(path_to_save, file_name_placement)

    with open(file_path_placement, 'w') as file:                                    # Write placement CSV (one row per letter)
        file.write('subject,letter,duration,coordinates\n')
        for record in placement_records:
            subject, letter, duration, coordinates = record
            file.write(f'{subject},{letter},{duration},{coordinates}\n')

    file_name_tracking = f"subject_{subject_nr}_tracking_{current_time}.csv"        # Trajectory samples filename
    file_path_tracking = os.path.join(path_to_save, file_name_tracking)

    with open(file_path_tracking, 'w') as file:                                     # Write tracking CSV (many samples per letter)
        file.write('subject,relative_drag_time,x,y,letter\n')
        for record in tracking_records:
            subject, relative_drag_time, x, y, letter = record
            file.write(f'{subject},{relative_drag_time},{x},{y},{letter}\n')

    def plot_tracking_data(file_path, subject_nr, save_path):                       # Chunk: read tracking CSV and save per-letter path plot
        tracking_data = pd.read_csv(file_path)
        subject_data = tracking_data[tracking_data['subject'] == subject_nr]

        plt.figure(figsize=(10, 8))
        for letter in subject_data['letter'].unique():
            letter_data = subject_data[subject_data['letter'] == letter]
            plt.plot(letter_data['x'], letter_data['y'],
                     marker='o', linestyle='-', label=f'Letter {letter}', markersize=2)

        plt.title(f'Tracking Data for Subject {subject_nr}')
        plt.xlabel('X Coordinate')
        plt.ylabel('Y Coordinate')
        plt.legend()
        plt.grid(True)
        plt.gca().invert_yaxis()                                                     # Match typical screen coordinates (y increases downward)

        image_path = os.path.join(save_path, f"subject_{subject_nr}_tracking_{current_time}.png")
        plt.savefig(image_path)
        plt.close()

    plot_tracking_data(file_path_tracking, subject_nr, path_to_save)                 # Generate trajectory plot PNG

    def capture_screenshot(canvas, screenshot_path):                                 # Chunk: screenshot helper using PsychoPy window frame capture
        window = self.experiment.window
        window.getMovieFrame()
        window.saveMovieFrames(screenshot_path)

    final_canvas = canvas(exp)                                                       # Chunk: draw final state screen (all letters + target line)
    final_canvas.clear()

    for key, letter in letters.items():
        color = '#00FF00' if letter['placed'] else 'white'
        final_canvas.text(letter['text'], center=True, x=letter['x'], y=letter['y'], color=color)

    final_canvas.line(-321, target_line_y, 321, target_line_y)
    final_canvas.show()

    screenshot_path = os.path.join(path_to_save, f"subject_{subject_nr}_final_screenshot_{current_time}.png")
    capture_screenshot(final_canvas, screenshot_path)                                # Save final screen screenshot

    print(f"Screenshot della schermata finale salvata in: {screenshot_path}")
