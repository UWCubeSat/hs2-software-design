## **2.1 \- Overview of Concept of Operations (CONOPS)**

The HuskySat-2 mission assumes a Low Earth Orbit (LEO) deployment with an orbital period of approximately 90 \[TBR\] minutes. When verifying the payload, the key constraints are power (see power budget), data storage capacity (see data budget), and ground station passes (see link budget). The satellite shall prioritize system health while completing all experiments. Experiments may be interrupted if any safe mode criteria is met.  

### **2.1.0 Terminology** 

* Phase: High level segment of the mission lifetime, on the order of days or weeks. Defined by major milestones or operational objectives. Encompasses multiple modes and states. HS2 includes deployment, bus commissioning phase, payload commissioning phase, experimentation phase, and decommissioning.   
* Mode: Operational configuration of the satellite. Responds to conditions within a phase via the satellite’s State Machine. It involves coordinated subsystem settings and includes functional states of each subsystem. HS2 includes safe mode, experiment mode, and standby mode.   
* State: a detailed, instantaneous configuration of individual subsystems or components within a mode. Defines specific parameter levels like power levels and data rates. States are the lowest level operational details, tabulated in a State Matrix.

### **2.1.1 \- Mission Operations Plan**

The Mission Operations Plan for HuskySat-2 outlines the operational framework to ensure successful execution of mission objectives: the demonstration of LOST and FOUND. This plan leverages the satellite’s orbit characteristics, ground station passes, and operational modes to maximize data collection, system health monitoring, and experiment validation, while adhering to UNP guidelines and the mission’s requirements. The plan is based on the satellite’s deployment in Low Earth Orbit (LEO).

### **2.1.2 Orbit Requirements** 

The orbit requirements are defined to ensure mission success, aligning with scientific objectives and launch vehicle constraints. These are preliminary and subject to refinement based on launch provider specifications.

| ORB-1 | The satellite shall operate in a Low Earth Orbit at altitude between 400km and 600km \[TBR\]. |
| :---- | :---- |
| ORB-2 | The satellite shall operate with an orbital eccentricity of less than 0.01 \[TBR\]. |
| ORB-3 | The satellite shall operate in an orbit with an inclination of at least 47.6 degrees \[TBR\]. |

The rationale for these requirements is that the altitude range balances power availability, communications capability, and visibility for cameras. It supports LOST’s star field imaging and FOUND’s Earth limb detection. Meanwhile, ORB-2 ensures consistent altitude for the experiment plan of testing LOST and FOUND (FOUND distance determination requires stable geometry) while also simplifying power management and thermal control. Finally, ORB-3 inclination range targets at least 47.6 degrees to ensure alignment with the ground station at University of Washington.

## **2.2 \- Mission Phases**

### **2.2.1 \- Deployment (\~ 5 min)**

1. Inhibit switches activated: deployment triggers power-on, activating EPS. After the inhibits are activated, the satellite will power on in a minimal power mode.   
2. Initial charge: Solar panels begin charging upon exposure to sunlight. The BMS (Battery management system) initiates MPPT to charge the battery.  
3. Power On Check: EPS verifies rail voltages and battery SOH using hardware logic. 

### **2.2.2 \- Bus Commissioning (\~1 week)**

1. CDH bootup: CDH boots flight computer  
   1. Run POST procedure:  
      1. Turn on the radio, immediately test for COMMS. if not working, reset, power cycle, etc.  
      2. IMMEDIATELY start beaconing  
      3. Test ADCS IMMU. If not working, reset, power cycle, switch to backup (if we have one, TBD)  
2. COMMS beaconing begins at \~45 minutes post deployment and transmits every TBD seconds the satellite has 2 patch antennas, enabling almost omnidirectional communications so that it is capable of closing a link even before detumbling.   
   1. **TODO: Does COMMS need GNSS during commissioning. Also when DO they need it?**  
3. Magnetorquer detumble: ADCS engages magnetorquers to dampen tumble rates from worst case scenario of 10 deg/s to experiment mode requirement, 0.5 deg/s, expected to take approximately 110 minutes.   
   1. This only happens after the batteries charge to a specific % charge, likely pretty high **TODO: Figure out number for this here**  
4. Solar panels deploy through burn wire mechanism (TODO: get more info)  
5. **Point solar panels at sun: magnetorquers orient satellite into optimal orientation, with solar panel face towards the sun**  
6. Solar panels charging: verify that the solar panels are generating power and charging the battery   
   1. Need some sort of FDIR here if possible (is it possible? How do we recover from not charging? Is it an ADCS issue?)  
   2. Is this redundant? Solar panels should already be charging if we got here (to some degree)  
      1. rewrite to focus more on checking whether ADCS pointed at the right place?  
7. Check out 2: Check Star Tracker operation. Confirm LOST’s payload hardware and software pipeline meet performance specs. CDH activates Sagitta and captures an image. Let Sagitta process the image.  
   1. FDIR procedure here: talk to payload  
8. Star tracker turns on for ADCS to set 'basepoint' attitude.  
9. Daily health and telemetry downlink   
   1. antennas are changing i got no idea rn  
10. Ground command to move onto Payload Commission phase

### **2.2.3 \- Payload Commissioning (\~1 week)**

**TODO: Do we charge to a specific % before we move on?**

Check out 1: Check ADCS Pointing. CDH commands ADCS to slew to known attitude (nadir or star field) using magnetorquers and sensor feedback.  

6. **TODO: Is this how it will ALWAYS HAPPEN\>???**

Check out 2: Check Star Tracker operation. Confirm LOST’s payload hardware and software pipeline meet performance specs. CDH activates Sagitta and captures an image. Let Sagitta process the image (why, what are we looking for?).

7. FDIR

Check out 3: CDH activates LOST camera and captures a starfield image. Process image through the pipeline to generate attitude quaternions.

8. See FDIR

Check out 3: Check FOUND camera works. Verify FOUND camera’s hardware and initial operation for horizon tracking. CDH powers on FOUND camera and initiates a test image capture of the Earth’s limb. Process the image to detect edges and log position data outputs. 

Check out 5: Test Downlink. Test ability to transmit images and data to ground station. Payload compresses images, sends to CDH, which sends to COMMS, which sends to ground.

Check out 4: Calibrate alignment. Get angle between LOST, FOUND, and ST optical axes. TBD on how

9. TBD but it seems like the best option is: take photo with sagitta and found camera. Downlink found camera and process attitude on Earth and compare to sagitta output attitude for alignment. We need a good way to process the photos and determine attitude *very* accurately  
   10. Create a full plan here (mini CONOPS?\!)

Required ground command to continue to experiment phase

### **2.2.4 \- Experiment Phase (\~6? weeks)**

See section 2.4: Experiment Plan   
 **AHHHHHHHHHHHHHHHHHHHHHHH**

### **2.2.5 \- Decommission** 

The decommission phase is initiated upon ground command for the experiment phase being completed. First, the satellite is sent into safe mode to halt payload operations and minimize power consumption. Then, the satellite naturally decays over time, targeting re-entry within 25 years per debris rules.

### **2.2.6 \- Mission Phases Tables** 

tbh don't know the purpose of this  
 0 \= off                  
 1 \= on  
 Table 6: Mission Phases Overview

| PHASE 0: DEPLOYMENT |  |  |  |  |  |  |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
|  | ADCS | CDH | PAY | EPS | COMMS | THR |
| Launch | 0 | 0 | 0 | 0 | 0 | 0 |
| Deployment | 0 | 0 | 0 | 1 | 0 | 0 |

| PHASE 1: BUS COMMISSIONING | ADCS | CDH | PAY | EPS | COMMS | THR |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| CDH Bootup | 0 | 1 | 0 | 1 | 1 | 0 |
| Detumble (MT) | 1 | 1 | 0 | 1 | 1 | 0 |
| Initial Charge | 1 | 1 | 0 | 1 | 1 | 1 |
| Downlink Status to Ground | 1 | 1 | 0 | 1 | 1 | 0 |

| PHASE 2: PAYLOAD COMMISSIONING | ADCS | CDH | PAY | EPS | COMMS | THR |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| CO.1 | 1 | 1 | 0 | 1 | 1 | 1 |
| CO.2 | 1 | 1 | 1 | 1 | 1 | 1 |
| CO.3 | 1 | 1 | 1 | 1 | 1 | 1 |
| CO.4 | 1 | 1 | 1 | 1 | 1 | 1 |
| CO.5 | 1 | 1 | 1 | 1 | 1 | 1 |

### **2.2.7- Mode Machine**

| Mode | Purpose | Entry Criteria | Exit Criteria |
| ----- | ----- | ----- | ----- |
| Experiment Mode | Conduct experiments on LOST and FOUND | \- Current Experiment is set by ground command (A-C) \- Must have at least 85% (User Guide 7.4.1.3 (TBR)) level of battery \- Check with sun sensor to ensure not facing sun | \- Battery below 50%\[TBR\] \-\> standby mode \- Data storage 90%\[TBR\] full \-\> standby mode \- Any safe mode entry criteria \-\>safe mode \- End of experiment \-\> Standby |
| Safe Mode | Low Power, Low complexity mode for emergencies and debugging | \- Satellite Reboots \- Battery drops Below 30\[TBR\]%  \- System fault encountered  \- Command from ground | \- Command from the ground \-\> promotion to standby \[TBR\] |
| Standby Mode (Eclipse) | Power conservation, Waiting for commands | \- Onboard Data Storage is 90%\[TBR\] full  \- Battery is below 50% TBR \- End of Experiment  \- In Eclipse (sun sensor) | \- Battery above 85% and data storage below 30%\[TBR\] full  \-\>experiment mode  \- Any safe mode entry criteria \-\>safe mode \- Exit eclipse \-\> Standby (sunlit) |
| Standby Mode (sunlit) | Charging and waiting for commands | \- Onboard Data Storage is 90%\[TBR\] full  \- Battery is below 50% TBR \- End of Experiment | \- Battery above 85% and data storage below 30%\[TBR\] full  \-\>experiment mode  \- Any safe mode entry criteria \-\>safe mode \- Eclipse \-\> Standby (Eclipse) |

## **2.3 \- Operational States** 

### **2.3.1 Satellite State Matrix**

|  |  |  |  |  |
| ----- | ----- | ----- | ----- | ----- |
| Subsystem | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| ADCS | On | On | On | On |
| Payload | Off | Off | Off | On |
| EPS | On | On | On | On |
| COM | On | On | On | On |
| CDH | On | On | On | On |
| Thermal | Off | On | Off | On |

### **2.3.2 ADCS State Matrix** 

Table 10: ADCS State Matrix

| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| ----- | ----- | ----- | ----- | ----- |
| IMMU | On | On | On | On |
| Sun sensors | On | On | On | On |
| Magnetorquers | On | Off | Off | On |

### **2.3.3 CDH State Matrix**

| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| ----- | ----- | ----- | ----- | ----- |
| Flight Computer | On | On | On | On |

### **2.3.4 EPS State Matrix**

| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| ----- | ----- | ----- | ----- | ----- |
| Solar Panels | On | On | On | On |
| BMS | On | On | On | On |
| PDS | On | On | On | On |

### **2.3.5 COMMS State Matrix**

|  |  |  |  |  |
| ----- | ----- | ----- | ----- | ----- |
| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| Transceiver | On \- Rx and downlinking during passes | On \- Rx on low, beaconing TBD | On \- Rx on low, beaconing TBD | On; full Rx and Tx during passes |
| Antennas | On | On | On | On |

### **2.3.6 Thermals State Matrix**

|  |  |  |  |  |
| ----- | ----- | ----- | ----- | ----- |
| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| Thermal Sensors (xTBD) | On | On | On | On |
| Passive MLI/radiators (TBD) | Passive | Passive | Passive | Passive |
| Kapton Heaters | On; if battery \<5 \[TBR\] celsius, Off if battery OK | On if batteries \<0C (priority) | On if batteries \<0 C (priority) | On if batteries \<0 C (priority) |

### **2.3.7 Payload State Matrix**

| Component | Standby (Sunlit) | Standby (Eclipse) | Safe | Experiment |
| ----- | ----- | ----- | ----- | ----- |
| LOST Camera | Off | Off | Off | On; taking star field images and forwarding to CDH |
| Star Tracker TM | On if ADCS requests attitude update; Off otherwise | Off | Off | On; taking star field images, calculating attitude, and sending data to CDH |
| GNSS | Off | Off | Off | On; taking timestamped measurements and sending to CDH |
| FOUND Camera | Off | Off | Off | On; Taking images of earths limb and sending to CDH |

## **2.4 \- Experiment Plan**

The following sections outline the different experiments planned to demonstrate the capabilities of the HS-2 payload, flowing down from the mission success criteria outlined in section 1.5. These experiments are intended to be completed as many times as outlined in the mission success criteria, and build upon each other in order of complexity (A is the simplest, then B, etc.). Any experiment that contains multiple success criteria counts towards each of them at the same time. These experiments will output data for error calculations, performed by the HS-2 ground team. 

### **Attitude Plan**

The experimentation phase leverages the Attitude Determination and Control Subsystem (ADCS) to orient the satellite, aligning the payload imaging hardware with their respective targets while adhering to operational constraints. The ADCS employs a three-axis stabilized control strategy using magnetorquers, pointing towards Earth’s limb for FOUND’s camera and a star field for LOST’s camera and the onboard star tracker. Attitude updates occur at TBD Hz, driven by the attitude algorithm. LOST’s camera, the Arcsec Sagitta, has an FOV of 25.4 degrees, allowing for visibility of the star field simultaneous to FOUND’s camera seeing Earth’s limb. 

#### **Horizon Detection for FOUND**

The satellite identifies the horizon for FOUND by relying on ADCS to maintain an Earth-pointing attitude, where the horizon appears as a distinct edge in the camera’s field of view. FOUND’s edge detection algorithm (CEDA/SEDA, Section 2.2.3) processes these images and isolates the Earth’s limb. Initial coarse pointing is achieved with GNSS data \[TBR\], while the star trackers provide fine attitude knowledge.

#### **Star Field Acquisition for LOST**

For LOST, the spacecraft achieves a clear view of the star field by slewing ADCS to a star-pointing attitude, avoiding Earth’s limb and sun interference. LOST’s centroiding (via center of gravity algorithm) requires at least 4 stars in its FOV, detected through a threshold-based star identification process (Tetra/Pyramid, 95% accuracy). LOST uses pre-loaded star catalogs and sensor’s real-time feedback to adjust pointing, ensuring unobstructed views during data acquisition.

## **2.5 Experiments**

## **2.6 Error Calculations**

