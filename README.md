import tkinter as tk
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
from matplotlib.widgets import Slider
import json
import os
from datetime import datetime

# from datetime import timedelta

# 1. Getting The file location.
current_directory = os.path.dirname(os.path.abspath(__file__))
data_filename = 'Data92J.txt'
data_filepath = os.path.join(current_directory, data_filename)
success_data_filename = 'SuccessTransactionData'
success_data_filepath = os.path.join(current_directory, success_data_filename)


# 2. Read the data of the file and annotate Success
def read_data_from_file(directory):
    data = []
    with open(directory, 'r') as file:
        for line in file:
            try:
                data_point = json.loads(line)
                timestamp = datetime.strptime(data_point['date'], '%Y-%m-%d %H:%M:%S.%f')
                bid = float(data_point['BID'])
                ask = float(data_point['ASK'])
                flag = data_point['FlagASKRSIRev'] == 'True'
                drow = data_point['drow']
                askrsi = float(data_point['ASKRSI'])
                askrsitip = float(data_point['ASKRSITip'])  # Access ASKRSITip with capitalized letters
                stdevb = float(data_point['STDevB'])  # Access STDevB with capitalized letters
                data.append({
                    'timestamp': timestamp,
                    'bid': bid,
                    'ask': ask,
                    'flag': flag,
                    'drow': drow,
                    'ASKRSI': askrsi,  # Capitalized 'ASKRSI'
                    'ASKRSITip': askrsitip,  # Capitalized 'ASKRSITip'
                    'STDevB': stdevb  # Capitalized 'STDevB'
                })
            except (ValueError, KeyError) as e:
                print(f"Error processing line: {line}. Error: {e}")
    return data


# 3. Read Success Transaction Data and Annotate, And Define the function "Update_Plot"
def read_success_transaction_data(directory):
    datarows = []
    with open(directory, 'r') as file:
        for line in file:
            data_point = json.loads(line)
            datarow = float(data_point['datarow'])
            formatted_datarow = float(
                format(datarow, ".2f"))  # Convert to string with 2 decimal places and then back to float
            datarows.append(formatted_datarow)
    return datarows


def clear_annotations():
    for annotation in ax.texts:
        annotation.set_visible(False)


def update_time_range(val):
    zoom_factor = int(scrollbar.val)
    new_initial_range = initial_range + zoom_factor
    if new_initial_range < 1:
        return

    start_idx = int(scrollbar.val)
    end_idx = start_idx + new_initial_range
    if end_idx > len(data):
        end_idx = len(data)
        start_idx = end_idx - new_initial_range

    start_time = data[start_idx]['timestamp']
    end_time = data[end_idx - 1]['timestamp']

    x_data = [entry['timestamp'] for entry in data[start_idx:end_idx]]  # Define x_data
    y_bid = [entry['bid'] for entry in data[start_idx:end_idx]]  # Define y_bid
    y_ask = [entry['ask'] for entry in data[start_idx:end_idx]]  # Define y_ask

    # Call the annotate_success_transactions function with success_datarows, start_time, and end_time
    annotate_success_transactions(success_datarows)
    bid_line.set_xdata(x_data)
    bid_line.set_ydata(y_bid)
    ask_line.set_xdata(x_data)
    ask_line.set_ydata(y_ask)

    ax.set_xlim(x_data[0], x_data[-1])
    ax.set_ylim(min(y_bid + y_ask), max(y_bid + y_ask))

    canvas.draw()


def annotate_success_transactions(success_datarows):
    clear_annotations()
    for datarow_index_str in success_datarows:
        datarow_index = int(datarow_index_str)
        if 0 <= datarow_index < len(data):
            data_point = data[datarow_index]
            timestamp = data_point['timestamp']
            entry_price = data_point['bid'] if data_point['flag'] else data_point['ask']
            side = 'BUY' if data_point['flag'] else 'SELL'
            ax.annotate(
                f'Success\nEntryPrice: {entry_price:.2f}\nSIDE:'
                f' {side}\ndatarow: {datarow_index}\ndatetime: {timestamp}',
                xy=(timestamp, entry_price),
                xytext=(-15, 20),
                textcoords='offset points',
                arrowprops=dict(facecolor='green', arrowstyle='->'),
                fontsize=8,
                color='green'
            )


# Two significant Update Plot in the Code. Update_plot, Update_main_plot
def update_plot(i):
    current_range_start = int(scrollbar.val)
    plt.cla()
    plt.plot(timestamps[current_range_start:i + 1], bids[current_range_start:i + 1], label='BID', color='blue')
    plt.plot(timestamps[current_range_start:i + 1], asks[current_range_start:i + 1], label='ASK', color='red')

    # Add arrows based on datarow within the current time range
    for datarow in flag_datarows:
        if datarow in drows:
            idx = drows.index(datarow)
            if current_range_start <= idx <= i:  # Only annotate points within the current time range
                annotation_text = "Datetime: {}\nDatarow: {}\nAsks: {}\nBids: {}".format(
                    timestamps[idx].strftime('%m-%d %H:%M:%S'), datarow, asks[idx], bids[idx])

                if show_askrsi_annotation:
                    annotation_text += "\nASKRSI: {}".format(data[idx]['ASKRSI'])
                if show_askrsitip_annotation:
                    annotation_text += "\nASKRSITip: {}".format(data[idx]['ASKRSITip'])
                if show_stdev_annotation:
                    annotation_text += "\nSTDevB: {}".format(data[idx]['STDevB'])

                plt.annotate(
                    annotation_text,
                    xy=(timestamps[idx], asks[idx]),
                    xytext=(timestamps[idx], asks[idx] - 5),
                    arrowprops=dict(facecolor='black', arrowstyle="->"),
                    fontsize=10)


show_askrsi_annotation = False
show_askrsitip_annotation = False
show_stdev_annotation = False


def debug_print_filtered_data(entry, min_value, max_value, selected_option):
    print("Filtered Data for Entry:")
    print("Selected Option:", selected_option)
    print("Selected Min:", min_value)
    print("Selected Max:", max_value)
    print("Timestamp:", entry['timestamp'])
    print("ASKRSI:", entry['ASKRSI'])  # Replace 'ASKRSI' with the actual key in your data
    print("Drow:", entry['drow'])
    print()
    print()


def update_main_plot(selected_option, min_value, max_value):
    global show_askrsi_annotation, show_askrsitip_annotation, show_stdev_annotation
    show_askrsi_annotation = selected_option == "ASKRSI"
    show_askrsitip_annotation = selected_option == "ASKRSITip"
    show_stdev_annotation = selected_option == "STDevB"

    # Rest of your function code...

    print("Selected Option:", selected_option)
    print("Min Value:", min_value)
    print("Max Value:", max_value)

    filtered_data = []

    for entry in data:
        value = entry.get(selected_option)
        if value is not None and min_value <= value <= max_value and value != 0:  # Exclude value == 0
            entry['flag'] = True  # Flag the entry if it meets the criteria
            filtered_data.append(entry)
            debug_print_filtered_data(entry, min_value, max_value, selected_option)
        else:
            entry['flag'] = False

    if not filtered_data:
        # Handle the case where no data meets the filtering criteria
        print("No data available for the selected filtering criteria.")
        return

    # Update the plot with the filtered data
    bid_line.set_xdata([entry['timestamp'] for entry in filtered_data])
    bid_line.set_ydata([entry['bid'] for entry in filtered_data])
    ask_line.set_xdata([entry['timestamp'] for entry in filtered_data])
    ask_line.set_ydata([entry['ask'] for entry in filtered_data])

    ax.set_xlim(filtered_data[0]['timestamp'], filtered_data[-1]['timestamp'])
    ax.set_ylim(min([entry['bid'] for entry in filtered_data] + [entry['ask'] for entry in filtered_data]),
                max([entry['bid'] for entry in filtered_data] + [entry['ask'] for entry in filtered_data]))

    # Annotate the filtered data based on the specified information
    clear_annotations()
    for entry in filtered_data:
        timestamp = entry['timestamp']
        entry_price = entry['bid'] if entry['flag'] else entry['ask']
        side = 'BUY' if entry['flag'] else 'SELL'
        askrsi = entry['ASKRSI']
        stdevb = entry['STDevB']
        askrsitip = entry['ASKRSITip']
        datarow = entry['drow']
        formatted_timestamp = timestamp.strftime('%Y-%m-%d %H:%M:%S')
        ax.annotate(
            f'Success\nSide: {side}\nEntry Price: {entry_price:.2f}\nASKRSI: {askrsi:.2f}\nSTDevb: {stdevb:.2f}'
            f'\nASKRSITip: {askrsitip:.2f}\nDatarow: {datarow}\nDate: {formatted_timestamp}',
            xy=(timestamp, entry_price),
            xytext=(-15, 20),
            textcoords='offset points',
            arrowprops=dict(facecolor='green', arrowstyle='->'),
            fontsize=8,
            color='green'
        )

    canvas.draw()


# 4. Call the Function of Success Transaction Data And Data.Txt
data = read_data_from_file(data_filepath)

# 5. Create Tkinter Window.
root = tk.Tk()
root.title("Data Analysis")

# 6. Create a Matplotlib Figure and Axes
fig, ax = plt.subplots(figsize=(12, 3))
fig_widgets, ax_widgets = plt.subplots(figsize=(8, 2.5))

# 7. Add labels, title, legend, etc.
ax.set_xlabel('Timestamp')
ax.set_ylabel('Price')
ax.set_title('ASK AND BID')
ax.grid(True)

# 8. Plot the Data
bid_line, = ax.plot([entry['timestamp'] for entry in data], [entry['bid'] for entry in data], label='BID', color='blue')
ask_line, = ax.plot([entry['timestamp'] for entry in data], [entry['ask'] for entry in data], label='ASK', color='red')
# Customize the appearance of the axes for your widgets
ax_widgets.set_title('Widget Controls')
ax_widgets.set_xlim(0, 1)  # Adjust the x-axis limits if needed
ax_widgets.set_ylim(0, 1)  # Adjust the y-axis limits if needed
ax_widgets.axis('off')  # Turn off axis for cleaner appearance

# 9. Show the Plot data
canvas = FigureCanvasTkAgg(fig, master=root)
canvas_widget = canvas.get_tk_widget()
canvas_widget.pack(side=tk.TOP, fill=tk.BOTH, expand=1)

canvas_widgets = FigureCanvasTkAgg(fig_widgets, master=root)
canvas_widgets_widget = canvas_widgets.get_tk_widget()
canvas_widgets_widget.pack(side=tk.TOP, fill=tk.BOTH, expand=1)

option_min_max = {
    "ASKRSI": (0.0, 50.0),
    "ASKRSITip": (0.0, 50.0),
    "STDevB": (0.0, 50.0)
}
# 10. Add the Slider for Min and Max.
ax_min_askrsi = plt.axes([0.2, 0.47, 0.1, 0.025], facecolor='lightgoldenrodyellow')
ax_max_askrsi = plt.axes([0.45, 0.47, 0.1, 0.025], facecolor='lightgoldenrodyellow')
ax_min_askrsitip = plt.axes([0.2, 0.45, 0.1, 0.025], facecolor='lightgoldenrodyellow')
ax_max_askrsitip = plt.axes([0.45, 0.45, 0.1, 0.025], facecolor='lightgoldenrodyellow')
ax_min_stdevb = plt.axes([0.2, 0.42, 0.1, 0.025], facecolor='lightgoldenrodyellow')
ax_max_stdevb = plt.axes([0.45, 0.42, 0.1, 0.025], facecolor='lightgoldenrodyellow')

s_min_askrsi = Slider(ax_min_askrsi, 'Min ASKRSI', 0, 50, valinit=0)
s_max_askrsi = Slider(ax_max_askrsi, 'Max ASKRSI', 0, 50, valinit=0)
s_min_askrsitip = Slider(ax_min_askrsitip, 'Min ASKRSITip', 00, 50, valinit=0)
s_max_askrsitip = Slider(ax_max_askrsitip, 'Max ASKRSITip', 00, 50, valinit=0)
s_min_stdevb = Slider(ax_min_stdevb, 'Min STDevB', 0, 50, valinit=0)
s_max_stdevb = Slider(ax_max_stdevb, 'Max STDevB', 0, 50, valinit=0)

var_min_askrsi = tk.StringVar()
var_max_askrsi = tk.StringVar()
var_min_askrsitip = tk.StringVar()
var_max_askrsitip = tk.StringVar()
var_min_stdevb = tk.StringVar()
var_max_stdevb = tk.StringVar()
# This is where the variable of slider stored.
current_min_askrsi = s_min_askrsi.val
current_max_askrsi = s_max_askrsi.val
current_min_askrsitip = s_min_askrsitip.val
current_max_askrsitip = s_max_askrsitip.val
current_min_stdevb = s_min_stdevb.val
current_max_stdevb = s_max_stdevb.val


def slider_callback(val):
    selected_option = selected_option_askrsi.get()  # Change this line based on the selected dropdown
    # Rest of your code...


    if selected_option == "ASKRSI":
        current_min_value = s_min_askrsi.val
        current_max_value = s_max_askrsi.val
    elif selected_option == "ASKRSITip":
        current_min_value = s_min_askrsitip.val
        current_max_value = s_max_askrsitip.val
    elif selected_option == "STDevB":
        current_min_value = s_min_stdevb.val
        current_max_value = s_max_stdevb.val
    else:
        current_min_value = 0
        current_max_value = 0

    # Pass success_datarows as an argument
    update_main_plot(selected_option, current_min_value, current_max_value)
    canvas.draw_idle()


s_min_askrsi.on_changed(slider_callback)
s_max_askrsi.on_changed(slider_callback)
s_min_askrsitip.on_changed(slider_callback)
s_max_askrsitip.on_changed(slider_callback)
s_min_stdevb.on_changed(slider_callback)
s_max_stdevb.on_changed(slider_callback)


# 11. Define the function for dropdowns.
# ... (Previous code)

def Dropdown_one(val):
    selected_option = selected_option_askrsi.get()
    var_min_askrsi.set(f'Min {selected_option}')
    var_max_askrsi.set(f'Max {selected_option}')
    current_min_value = s_min_askrsi.val
    current_max_value = s_max_askrsi.val

    # Make the selected option unselectable in the other dropdowns

    update_main_plot(selected_option, current_min_value, current_max_value)
    canvas.draw_idle()


def DropDown_Two(val):
    selected_option = selected_option_askrsitip.get()
    var_min_askrsitip.set(f'Min {selected_option}')
    var_max_askrsitip.set(f'Max {selected_option}')
    current_min_value = s_min_askrsitip.val
    current_max_value = s_max_askrsitip.val

    # Make the selected option unselectable in the other dropdowns

    update_main_plot(selected_option, current_min_value, current_max_value)
    canvas.draw_idle()


def Dropdown_Three(val):
    selected_option = selected_option_stdevb.get()
    var_min_stdevb.set(f'Min {selected_option}')
    var_max_stdevb.set(f'Max {selected_option}')
    current_min_value = s_min_stdevb.val
    current_max_value = s_max_stdevb.val

    # Make the selected option unselectable in the other dropdowns

    update_main_plot(selected_option, current_min_value, current_max_value)
    canvas.draw_idle()


selected_option_askrsi = tk.StringVar(root)
selected_option_askrsi.set("None")  # Default empty choice

selected_option_askrsitip = tk.StringVar(root)
selected_option_askrsitip.set("None")  # Default empty choice

selected_option_stdevb = tk.StringVar(root)
selected_option_stdevb.set("None")  # Default empty choice
def update_slider(selected_option):
    var_min_slider.set(f"Min {selected_option}")
    var_max_slider.set(f"Max {selected_option}")
    s_min_slider.valmin, s_max_slider.valmax = option_min_max[selected_option]
    s_min_slider.valinit, s_max_slider.valinit = option_min_max[selected_option]
    s_min_slider.reset()
    s_max_slider.reset()
    slider_callback(None)

def dropdown_changed(selected_option):
    var_min_slider.set(f"Min {selected_option}")
    var_max_slider.set(f"Max {selected_option}")
    s_min_slider.valmin = option_min_max[selected_option][0]
    s_min_slider.valmax = option_min_max[selected_option][1]
    s_max_slider.valmin = option_min_max[selected_option][0]
    s_max_slider.valmax = option_min_max[selected_option][1]
    s_min_slider.valinit = option_min_max[selected_option][0]
    s_max_slider.valinit = option_min_max[selected_option][1]
    s_min_slider.reset()
    s_max_slider.reset()
    slider_callback(None)


def refresh():
    global data
    data = read_data_from_file(data_filepath)  # Reload data from file
    success_datarows = read_success_transaction_data(success_data_filepath)  # Reload success data
    slider_callback(None)  # Reset the plot and annotations
    update_time_range(0)   # Reset the time range
    update_left_right(0)   # Reset the left-right scroll

# Create the refresh button
refresh_button = tk.Button(root, text="Refresh", command=refresh)
refresh_button.place(x=10, y=900)

def update_left_right(val):
    scroll_amount = int(left_right_slider.val)
    start_idx = scroll_amount
    end_idx = start_idx + initial_range

    if end_idx > len(data):
        end_idx = len(data)
        start_idx = end_idx - initial_range

    x_data = [entry['timestamp'] for entry in data[start_idx:end_idx]]
    y_bid = [entry['bid'] for entry in data[start_idx:end_idx]]
    y_ask = [entry['ask'] for entry in data[start_idx:end_idx]]

    clear_annotations()
    annotate_success_transactions(success_datarows)

    bid_line.set_xdata(x_data)
    bid_line.set_ydata(y_bid)
    ask_line.set_xdata(x_data)
    ask_line.set_ydata(y_ask)

    ax.set_xlim(x_data[0], x_data[-1])
    ax.set_ylim(min(y_bid + y_ask), max(y_bid + y_ask))

    canvas.draw()


# 12. Put the Dropdowns Widget.
dropdown_options = ["None", "ASKRSI", "ASKRSITip", "STDevB"]
selected_option_askrsi = tk.StringVar(root)
selected_option_askrsi.set("None")
askrsi_dropdown = tk.OptionMenu(root, selected_option_askrsi, *dropdown_options, command=dropdown_changed)
askrsi_dropdown.place(x=625, y=900)

selected_option_askrsitip = tk.StringVar(root)
selected_option_askrsitip.set("None")
askrsitip_dropdown = tk.OptionMenu(root, selected_option_askrsitip, *dropdown_options, command=dropdown_changed)
askrsitip_dropdown.place(x=625, y=920)

selected_option_stdevb = tk.StringVar(root)
selected_option_stdevb.set("None")
stdevb_dropdown = tk.OptionMenu(root, selected_option_stdevb, *dropdown_options, command=dropdown_changed)
stdevb_dropdown.place(x=625, y=940)


def askrsi_checkbox_callback():
    global show_askrsi_annotation
    show_askrsi_annotation = not show_askrsi_annotation
    update_plot(len(timestamps) - 1)


def askrsitip_checkbox_callback():
    global show_askrsitip_annotation
    show_askrsitip_annotation = not show_askrsitip_annotation
    update_plot(len(timestamps) - 1)


def stdev_checkbox_callback():
    global show_stdev_annotation
    show_stdev_annotation = not show_stdev_annotation
    update_plot(len(timestamps) - 1)


askrsi_checkbox = tk.Checkbutton(root, text="ASKRSI", command=askrsi_checkbox_callback)
askrsi_checkbox.place(x=125, y=900)

askrsitip_checkbox = tk.Checkbutton(root, text="ASKRSITip", command=askrsitip_checkbox_callback)
askrsitip_checkbox.place(x=125, y=920)

stdev_checkbox = tk.Checkbutton(root, text="STDevB", command=stdev_checkbox_callback)
stdev_checkbox.place(x=125, y=940)

success_datarows = read_success_transaction_data(success_data_filepath)

# Call the callback function to initialize the plot
slider_callback(None)

# 13. Create a slider for zoom in and zoom out of Timestamp and Plot.
initial_range = 100
scrollbar_ax = plt.axes([0.7, 0.009, 0.2, 0.01], facecolor="lightgoldenrodyellow")
scrollbar = Slider(scrollbar_ax, 'Time Range', 0, len(data) - initial_range, valinit=4062)
# 14. Create a slider for left to right
left_right_slider_ax = plt.axes([0.7, 0.04, 0.2, 0.01], facecolor="lightgoldenrodyellow")
left_right_slider = Slider(left_right_slider_ax, 'Left-Right Scroll', 0, len(data) - initial_range, valinit=0)
left_right_slider.on_changed(update_left_right)

# This is for the function Annotate Function. So that function can access the exact file SuccessTransactionData.
annotate_success_transactions(success_datarows)
update_time_range(0)  # Call this function to initialize the annotations based on the initial time range
update_left_right(0)

# This function is for updating the time_range while sliding the scroll bar.
scrollbar.on_changed(update_time_range)
root.mainloop()
