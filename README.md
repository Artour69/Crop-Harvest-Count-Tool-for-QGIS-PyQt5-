from PyQt5.QtWidgets import QWidget, QVBoxLayout, QComboBox, QPushButton, QLabel
from datetime import datetime
import calendar
from qgis.core import QgsProject

# List of crops
crops = ['AMPALAYA', 'EGGPLANT (TALONG)', 'OKRA', 'TOMATO (KAMATIS)', 'SWEET POTATO', 'BITTER GOURD', 
         'CORN', 'CABBAGE (REPOLYO)', 'STRING BEANS', 'PECHAY', 'CUCUMBER (PIPINO)']

# Estimated yield per unit area (in kg per hectare)
yield_per_unit_area = {
    'AMPALAYA': 3000,
    'EGGPLANT (TALONG)': 2500,
    'OKRA': 2000,
    'TOMATO (KAMATIS)': 4000,
    'SWEET POTATO': 3500,
    'BITTER GOURD': 2200,
    'CORN': 6000,
    'CABBAGE (REPOLYO)': 2800,
    'STRING BEANS': 2500,
    'PECHAY': 1500,
    'CUCUMBER (PIPINO)': 2000
}

# Global window and combo box references
window = None
crop_combo = None
month_combo = None
year_combo = None
result_label = None

# Function to clear memory and avoid crashes
def clear_memory():
    global crop_layer, plot_layer
    crop_layer = None
    plot_layer = None

# Function to count crops based on selected month and year
def count_crops():
    clear_memory()  # Clear previous layer references

    selected_month_name = month_combo.currentText()
    selected_year = int(year_combo.currentText())
    selected_crop = crop_combo.currentText()

    selected_month = list(calendar.month_name).index(selected_month_name)

    # Select the correct layers
    crop_layers = QgsProject.instance().mapLayersByName('1')
    plot_layers = QgsProject.instance().mapLayersByName('Plot Information Table')

    if not crop_layers or not plot_layers:
        result_label.setText("Cannot find '1' (Crop Cycle) or 'Plot Information Table'.")
        return
    
    crop_layer = crop_layers[0]
    plot_layer = plot_layers[0]

    # Initialize counters and data structures
    crop_harvest_count = 0
    total_yield = 0.0
    harvesters_info = {}

    area_info = {}
    for plot_feature in plot_layer.getFeatures():
        plot_id = plot_feature['ID']
        if not plot_id:
            continue

        try:
            AREA = float(plot_feature['AREA'])
        except (ValueError, TypeError):
            continue

        owner_tenant = plot_feature['owner/tenant']
        barangay = plot_feature['brgy']
        area_info[plot_id] = {'AREA': AREA, 'owner': owner_tenant, 'barangay': barangay}

    for feature in crop_layer.getFeatures():
        try:
            crop_planted = feature['CROP_PLANTED']
            date_of_harvest = feature['DATE_OF_HARVEST']
        except KeyError:
            continue

        plot_id = feature['PLOT_ID']
        if plot_id not in area_info:
            continue

        AREA = area_info[plot_id]['AREA']
        owner_tenant = area_info[plot_id]['owner']
        barangay = area_info[plot_id]['barangay']

        if date_of_harvest:
            try:
                harvest_date = datetime.strptime(date_of_harvest, '%m/%d/%Y')

                if crop_planted.strip().upper() == selected_crop.strip().upper() and harvest_date.month == selected_month and harvest_date.year == selected_year:
                    crop_harvest_count += 1
                    estimated_yield = AREA * yield_per_unit_area.get(crop_planted.strip().upper(), 0)
                    total_yield += estimated_yield

                    if owner_tenant in harvesters_info:
                        harvesters_info[owner_tenant]['yield'] += estimated_yield
                    else:
                        harvesters_info[owner_tenant] = {'barangay': barangay, 'yield': estimated_yield}

            except ValueError:
                continue

    harvesters_list = "\n".join([f"{owner} from {info['barangay']}: {info['yield']:.2f} kg" for owner, info in harvesters_info.items()])
    harvesters_list = harvesters_list if harvesters_list else "No harvesters found."

    result_label.setText(f"Total {selected_crop} crops to be harvested in {selected_month_name} {selected_year}: {crop_harvest_count}\n"
                         f"Total Estimated Yield: {total_yield:.2f} kg\n"
                         f"Harvesters:\n{harvesters_list}")

# Function to close the window without crashing QGIS
def close_app():
    global window
    if window:
        window.close()
        window = None  # Reset the global window reference

def create_window():
    global window, crop_combo, month_combo, year_combo, result_label

    if window is not None:
        window.close()  # Close any existing window before creating a new one
        window = None

    # Create a new window
    window = QWidget()
    window.setWindowTitle("Crop Harvest Count")

    layout = QVBoxLayout()

    # Create a dropdown for selecting the crop
    crop_combo = QComboBox()
    crop_combo.addItems(crops)

    # Create a dropdown for selecting the month (by name)
    month_combo = QComboBox()
    month_names = list(calendar.month_name)[1:]
    month_combo.addItems(month_names)

    # Create a dropdown for selecting the year
    year_combo = QComboBox()
    years = [str(i) for i in range(2020, 2031)]
    year_combo.addItems(years)

    # Create a button to trigger the count
    count_button = QPushButton("Count Crops")

    # Create a label to show the result
    result_label = QLabel("Total crops to be harvested: ")

    # Create a close button to safely exit the app
    close_button = QPushButton("Close")

    # Add widgets to the layout
    layout.addWidget(QLabel("Select Crop:"))
    layout.addWidget(crop_combo)
    layout.addWidget(QLabel("Select Harvest Month:"))
    layout.addWidget(month_combo)
    layout.addWidget(QLabel("Select Harvest Year:"))
    layout.addWidget(year_combo)
    layout.addWidget(count_button)
    layout.addWidget(result_label)
    layout.addWidget(close_button)

    window.setLayout(layout)

    # Connect the button click to the functions
    count_button.clicked.connect(count_crops)
    close_button.clicked.connect(close_app)

    # Show the window
    window.show()

# Create the window
create_window()
