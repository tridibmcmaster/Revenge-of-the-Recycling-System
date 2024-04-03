# Revenge-of-the-Recycling-System
#Final Virtual Environment Code 
ip_address = 'localhost' # Enter your IP Address here
project_identifier = 'P3B' # Enter the project identifier i.e. P2A or P2B

# SERVO TABLE CONFIGURATION
short_tower_angle = 315 # enter the value in degrees for the identification tower
tall_tower_angle = 90 # enter the value in degrees for the classification tower
drop_tube_angle = 180 # enter the value in degrees for the drop tube. clockwise rotation from zero degrees

# BIN CONFIGURATION
# Configuration for the colors for the bins and the lines leading to those bins.
# Note: The line leading up to the bin will be the same color as the bin

bin1_offset = 0.20 # offset in meters
bin1_color = [1,0,0] # e.g. [1,0,0] for red
bin1_metallic = False

bin2_offset = 0.20
bin2_color = [0,1,0]
bin2_metallic = False

bin3_offset = 0.20
bin3_color = [0,0,1]
bin3_metallic = False

bin4_offset = 0.20
bin4_color = [1, 1, 1]
bin4_metallic = False
#--------------------------------------------------------------------------------
import sys
sys.path.append('../')
from Common.simulation_project_library import *

hardware = False
if project_identifier == 'P3A':
    table_configuration = [short_tower_angle,tall_tower_angle,drop_tube_angle]
    configuration_information = [table_configuration, None] # Configuring just the table
    QLabs = configure_environment(project_identifier, ip_address, hardware,configuration_information).QLabs
    table = servo_table(ip_address,QLabs,table_configuration,hardware)
    arm = qarm(project_identifier,ip_address,QLabs,hardware)
else:
    table_configuration = [short_tower_angle,tall_tower_angle,drop_tube_angle]
    bin_configuration = [[bin1_offset,bin2_offset,bin3_offset,bin4_offset],[bin1_color,bin2_color,bin3_color,bin4_color],[bin1_metallic,bin2_metallic, bin3_metallic,bin4_metallic]]
    configuration_information = [table_configuration, bin_configuration]
    QLabs = configure_environment(project_identifier, ip_address, hardware,configuration_information).QLabs
    table = servo_table(ip_address,QLabs,table_configuration,hardware)
    arm = qarm(project_identifier,ip_address,QLabs,hardware)
    bot = qbot(0.1,ip_address,QLabs,project_identifier,hardware)
#--------------------------------------------------------------------------------
# STUDENT CODE BEGINS
#---------------------------------------------------------------------------------



#Get the bin_id as an integer, used for indexing
def get_bin_id(bin_id):
    bin_number = bin_id.split("0")[1]
    return int(bin_number)


#Collect information about the dispensed container's attributes
def get_container_info(container):
    duration = 1
    #outputs the material, mass, and bin ID for the input container as a 3-item list.
    container_info = table.dispense_container(container, True)
    print("Container ID: ", container)
    print("Container material: ", container_info[0])
    print("Container mass (in grams): ", container_info[1])
    print("Container destination: ", container_info[2])
    return container_info

#Determines the path that the Q-arm should follow to pick up and drop off a container properly.
#start refers to the starting point to grab container.
#end refers to the container's drop-off location to the hopper.
#count is an integer that refers to the location where the container should be dropped off based on how many containers were previously placed there.
def get_qarm_movement_path(start, end, count):
    #x_start, y_start, z_start are x, y, and z coordinates of the starting point location respectively.
    x_start = start[0]
    y_start = start[1]
    z_start = start[2]

    #x_end, y_end, z_end are x, y, and z coordinates of the drop-off point location respectively.
    x_end = end[0]
    y_end = end[1]
    z_end = end[2]

    #Move arm to pick up a container.
    time.sleep(1)
    arm.move_arm(x_start, y_start, z_start)
    time.sleep(1)
    arm.control_gripper(40)
    time.sleep(1)
    arm.rotate_shoulder(-5)
    time.sleep(1)
    arm.rotate_base(-20)
    time.sleep(1)
    arm.rotate_shoulder(-45)
    time.sleep(1)
    arm.rotate_base(-70)
    time.sleep(1)


    #Determines exactly where to place the container in the hopper depending on how many containers were previously placed there.
    arm.rotate_base(-12*count)
    time.sleep(1)
    arm.rotate_shoulder(25)

    #Moves the arm to the hopper.
    time.sleep(1)
    arm.move_arm(x_end, y_end, z_end)
    time.sleep(2)
    arm.control_gripper(-15)
    time.sleep(2)
    arm.rotate_shoulder(-6)
    time.sleep(1)
    arm.rotate_shoulder(-30)

    #Returns to home position.
    arm.home()
    return

#Collects the container from servo table and drops it off to the hopper.
def place_qarm(container, count):
    container_gap = 0.08 #distance between two containers in x-axis in the hopper.
    hopper = [0, -0.615, 0.53] #position of hopper.

    hopper[0] -= container_gap * count

    tall = [0.65, 0, 0.28]
    short = [0.65, 0, 0.23]

    #Determine the pick-up location depending on the height of container.        
    if container == 2 or container == 5:
        destination = short
    else:
        destination = tall

    get_qarm_movement_path(destination, hopper, count)
    return


#Controls the movement of the hopper. Speed is a list with two items.
def move_hopper(speed):
    #Activate linear actuator.
    bot.activate_linear_actuator()
    normal_speed = speed[0]
    turning_speed = speed[1]

    #Rotates hopper to deposit container inside the destination bin.
    time.sleep(2)
    bot.rotate_hopper(45)
    time.sleep(2)
    bot.rotate_hopper(0)
    time.sleep(1)

    return

#Transfer of Qbot following yellow line.
def move_qbot(start_time, deposited, speed, bin_id, color_sensor_info, home):
    #Determines when to apply color sensors.
    current_time = time.time()
    time_elapsed = round(current_time - start_time, 1)

    bot.activate_color_sensor()  
    color_sensor_info = bot.read_color_sensor()

    normal_speed = speed[0]
    turning_speed = speed[1]
   
    #Activates IR sensor and collect data.
    bot.activate_line_following_sensor()
    left_IR2 = bot.line_following_sensors()[0]
    right_IR2 = bot.line_following_sensors()[1]
    bot_location = bot.position()

    #Coordinates of home location.
    home_location = [1.5, 0, 0]

    #bot location in quanser virtual environment (only cosidering x and y coordinates, since bot never moves in z direction).
    x_bot = bot_location[0]
    y_bot = bot_location[1]

    x_distance_from_home = abs(x_bot - home_location[0])
    y_distance_from_home = abs(y_bot - home_location[1])

    bin_colors = [bin1_color, bin2_color, bin3_color, bin4_color]

    #Corresponding target bin's number in integer.
    bin_number = get_bin_id(bin_id)

    #Target color of corresponding bin.
    target_color = bin_colors[bin_number - 1]
    print("Target bin number: ", bin_number)
    print("Target bin color: ", target_color)

    #Distance between bot's location to target bin's location in x and y coordinates.

    print("Color Sensor info:  ", color_sensor_info)

    #Updates the color sensor information every 0.5 milliseconds.
    if time_elapsed % 0.2 == 0:
        color_sensor_info = bot.read_color_sensor()

    if bin_number == 1 and color_sensor_info[0][0] == 1 and color_sensor_info[0][1] == 0 and color_sensor_info[0][2] == 0 and not(deposited):
        bot.forward_time(2.7)
        bot.rotate(-33)
        bot.stop()
        print("The target bin color 'Red' is detected.")
        move_hopper(speed)
        deposited = True
        speed = [0.1, 0.1]
       
    if bin_number == 2 and color_sensor_info[0][0] == 0 and color_sensor_info[0][1] == 1 and color_sensor_info[0][2] == 0 and not(deposited):
        bot.forward_time(2)
        bot.stop()
        print("The target bin color 'Green' is detected.")
        move_hopper(speed)
        deposited = True
        speed = [0.1, 0.1]

    if bin_number == 3 and color_sensor_info[0][0] == 0 and color_sensor_info[0][1] == 0 and color_sensor_info[0][2] == 1 and not(deposited):
        bot.forward_time(2.8)
        bot.rotate(-24)
        bot.stop()
        print("The target bin color 'Blue' is detected.")
        move_hopper(speed)
        deposited = True
        speed = [0.1, 0.1]

    if bin_number == 4 and color_sensor_info[0][0] == 1 and color_sensor_info[0][1] == 1 and color_sensor_info[0][2] == 1 and not(deposited):
        bot.forward_time(2)
        bot.stop()
        print("The target bin color 'White' is detected.")
        move_hopper(speed)
        deposited = True
        speed = [0.1, 0.1]
   

    else:
        #If the deposit process is completed, proceed to return home.
        print("Distance from home: ", x_distance_from_home, y_distance_from_home, deposited)
        if x_distance_from_home < 0.05 and y_distance_from_home < 0.05 and deposited:
            bot.stop()
            print("Returned home")
            print("Current bot position: ", bot.position())
            home = True
            return deposited, speed, home

        #Find and follow the yellow line.
        else:
            bot.activate_line_following_sensor()
            print(home, deposited)
            if left_IR2 == 1 and right_IR2 == 1:
                bot.set_wheel_speed([normal_speed, normal_speed])
                print("straight")
            elif left_IR2 == 1 and right_IR2 == 0:
                bot.set_wheel_speed([turning_speed/4, turning_speed/2])
                print("turning left")
            elif left_IR2 == 0 and right_IR2 == 1:
                bot.set_wheel_speed([turning_speed/2, turning_speed/4])
                print("turning right")
            else:
                bot.set_wheel_speed([0,0])
                bot.rotate(5)
                print("The yellow line is lost. Attempting to find line")

    return deposited, speed, home


def main():
    deposited = False
    home = False
    starting_bin = None
    speed = [0.1, 0.1]
    start_time = time.time()
    bot_location = bot.position()
    #Activate color sensor.
    bot.activate_color_sensor()

    color_sensor_info = bot.read_color_sensor()

    #Initialize variables.
    hopper_weight = 0
    hopper_count = 0
    hopper_bin = 0
    cycles = 0
    container_info = None

    while True:
        if cycles > 0:          
            bot.forward_distance(0.14)
            time.sleep(1)
            bot.rotate(92)
            time.sleep(1)

        if cycles == 0:          
            bot.forward_distance(0.14)
            bot.rotate(92)

        #A maximum of three containers are allowed to put in the hopper per cycle.
        while hopper_count < 3:
            if cycles == 0 or hopper_count > 0:
                #A random container id will be output as an integer between 1 and 6.
                random_container = random.randint(1, 6)
                container_info = get_container_info(random_container)
            material = container_info[0]
            weight = container_info[1]
            bin_id = container_info[2]

            #Extracts the bin number as an integer.
            bin_id_integar = get_bin_id(bin_id)

            if hopper_weight + weight <= 90:

                #Applicable for the first container to be placed in the hopper.
                if hopper_count == 0:
                    hopper_bin = bin_id
                #Applicable if the subsequent container also goes to the same destination bin.
                if hopper_bin == bin_id:
                    place_qarm(random_container, hopper_count)
                    #Increment of hopper weight as a new container is loaded.
                    hopper_weight += weight
                    hopper_count += 1
                #Applicable if the subsequent container has a different destination bin.
                else:
                    print("Different target bin")
                    cycles += 1
                    break

            else:
                print("The total load on the hopper cannot exceed 90 grams")
                cycles += 1
                break
           
                           
        bot.rotate(-92)
        print("Servo table and qarm: done")
        while True:
            deposited, speed, home = move_qbot(start_time, deposited, speed, hopper_bin, color_sensor_info, home)

            if home:
                break
        print("qbot:done")

        #Bot stops for one second. Then, it runs the next cycle.
        next_cycle = time.sleep(1)
        if next_cycle == time.sleep(1):
            #Reset values for another run.
            hopper_weight = 0
            hopper_count = 0
            hopper_bin = 0
            deposited = False
            home = False

        else:
            print("All Done!")
            return
    return

main()
