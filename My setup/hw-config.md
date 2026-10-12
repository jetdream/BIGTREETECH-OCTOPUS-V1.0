 * Spindle & Laser control
 *
 * Add the M3, M4, and M5 commands to turn the spindle/laser on and off, and
 * to set spindle speed, spindle direction, and laser power.

  #define SPINDLE_LASER_ENA_PIN PE5    // FAN1_PIN
  #define SPINDLE_LASER_PWM_PIN PD12   // FAN2_PIN


  /**
   * Speed / Power can be set ('M3 S') and displayed in terms of:
   *  - PWM255  (S0 - S255)
   */
  #define CUTTER_POWER_UNIT PWM255

  LASER_POWER_INLINE is DISABLED

  G0 does not turn laser powered off (LASER_MOVE_POWER param)
