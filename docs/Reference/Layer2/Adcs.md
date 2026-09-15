## ADCS Hardware Managers

The ADCS hardware-manager layer owns sensor and actuator protocols for attitude determination and control. Each manager follows the shared initialization and recovery pattern while `AdcsApplication` selects mission behavior and control targets.

The following component designs cover the IMU, sun sensors, and magnetorquer PWM control.
