###############################################################################
# Input transition constraints
###############################################################################
#FAST I2C
set SCL_PERIOD 1000
#SDA period is put to 1000 but in real life it will be bigger up to 2*1000, to restudy in case of setup violation
set SDA_PERIOD 1000
set TSU_STA 260
set TSU_DAT 50
#THD_DAT min=0 but to avoid any unwanted restart, I made the hold bigger
set THD_DAT -2 
set THD_STA 260
set TSU_STO 260
set TBUF 500

# VDD corner
set MIN_VDD 0.7

# Rise transitions
set SCL_TR_RISE 0.0003835223
set SDA_TR_RISE 0.0003835223

# Fall transitions
set SCL_TR_FALL 0.0003699792

set SDA_TR_FALL 0.0003699792

# Apply transitions
set_input_transition -rise $SCL_TR_RISE [get_ports scl]
set_input_transition -fall $SCL_TR_FALL [get_ports scl]

set_input_transition -rise $SDA_TR_RISE [get_ports sda_in]
set_input_transition -fall $SDA_TR_FALL [get_ports sda_in]


# Output load 
set_load -max 0.0451191 [get_ports sda_out]
set_load -max 0.0451191 [get_ports o_reg_*]
set_load -min 0.000207 [get_ports sda_out]
set_load -min 0.000207 [get_ports o_reg_*]

set_dont_touch [get_cells u_sda_in_dat_buf]
set_dont_touch [get_cells u_scl_in_start_dat_buf]
set_dont_touch [get_cells u_scl_in_stop_dat_buf]
set_dont_touch [get_cells u_rstn_start_stop_async_buf]
set_dont_touch [get_cells u_rstn_fsm_async_buf]


#required sdc
#setup data
create_clock -name i2c_scl_clk -period $SCL_PERIOD -add [get_ports scl]
create_clock -name sdi_clk -period $SDA_PERIOD [get_ports sda_in]

#Enable Reset checks, Dummy value
set_input_delay -clock "i2c_scl_clk" 10.0 [get_ports rst_n]

# Associate a unique pin relative to sda_in for Data sampling by SCL
set_input_delay -clock "i2c_scl_clk"  -max [expr $SCL_PERIOD - $TSU_DAT] [get_pins u_sda_in_dat_buf/X]
set_input_delay -clock "i2c_scl_clk" -add_delay  -min $THD_DAT [get_pins u_sda_in_dat_buf/X] 

# Port SCL is associated to i2c_scl_clk since the create_clock constraint, the 4 constraints below break the SCL from being reported 
# in the -to [get_clocks sdi_clk]  reports and instead use unique pins.

#FF sampled by falling edge
set_input_delay -clock [get_clocks sdi_clk] -max [expr $SDA_PERIOD - $TSU_STA] -clock_fall [get_pins u_scl_in_start_dat_buf/X]
set_input_delay -clock [get_clocks sdi_clk] -min $THD_STA -clock_fall [get_pins u_scl_in_start_dat_buf/X] 

#FF sampled by rising edges
set_input_delay -clock [get_clocks sdi_clk] -max [expr $SDA_PERIOD - $TSU_STO] [get_pins u_scl_in_stop_dat_buf/X]
set_input_delay -clock [get_clocks sdi_clk] -min $TBUF [get_pins u_scl_in_stop_dat_buf/X] 

#Explicit stopping of clocks when used as data => No real effect on the reports really 
set_clock_sense -stop_propagation [get_pins u_sda_in_dat_buf/X]
set_clock_sense -stop_propagation [get_pins u_scl_in_start_dat_buf/X]

set ACTIVATE_CUSTOM_RESET_CHECK 0
if{$ACTIVATE_CUSTOM_RESET_CHECK} {
    #!!!!!!Some how dummy but should be monitored (Theos set_m*_delay is to tune if needed)
    #because the start and stop are associated above to the sda clock, they are removed from any reset deassertion checks
    #because in the design I used them as part of reset async, so set_max_delay force the two Q pins to become again start_point
    #set_max_delay $THD_STA -from [get_pins start_reg/Q] -to [get_pins */RD]
    #set_min_delay 0 -from [get_pins start_reg/Q] -to [get_pins */RD]
    #set_max_delay [expr $TBUF + $THD_DAT] -from [get_pins stop_reg/Q] -to [get_pins */RD]
    #set_min_delay 0 -from [get_pins stop_reg/Q] -to [get_pins */RD]
    
    #set_max_delay $THD_STA -to [get_pins start_reg/RD]
    #set_min_delay 0 -to [get_pins start_reg/RD]
    #set_max_delay [expr $TBUF + $THD_DAT] -to [get_pins stop_reg/RD]
    #set_min_delay 0 -to [get_pins stop_reg/RD] 
    
    # clk2 -> clk1
    
    #set_data_check 0 \
       #-from [get_pins start_reg/Q] \
       #-to [get_clocks i2c_scl_clk]
    
    #set_data_check 0 \
       #-from [get_pins start_reg/Q] \
       #-to [get_clocks i2c_scl_clk]
    
    ## clk1 -> clk2
    
    #set_data_check 0  \
       #-from [get_pins stop_reg/Q] \
       #-to [get_clocks sdi_clk]
    
    #set_data_check 0  \
       #-from [get_pins stop_reg/Q] \
       #-to [get_clocks sdi_clk]
}

#Use  the  set_clock_groups  or set_false_path command to disable unwanted clock combinations.
#Dummy value, like imagining SCL will sample but it depends on the clock of the registers block mainly
set_output_delay -clock i2c_scl_clk [expr $SCL_PERIOD/2] [get_ports o_reg_*]
set_output_delay -clock i2c_scl_clk [expr $SCL_PERIOD/4] [get_ports sda_out]

#report_port -verbose [get_ports sda_in]
#report_timing \
  #-delay_type max \
  #-max_paths 10 \
  #-slack_lesser_than 999999 \
  #-to [all_registers]


  #check sanity info
  report_port -verbose [get_ports *]
  #These two commands should report that they are not constrained as replaced by pins in the input_delay constraints
  report_timing   -delay_type min   -max_paths 10    -slack_lesser_than 999999   -from [get_ports scl]
  report_timing   -delay_type min   -max_paths 10   -slack_lesser_than 999999  -from [get_ports sda_in]
  report_timing   -delay_type max   -max_paths 10   -slack_lesser_than 999999  -to start_reg/D
  report_timing   -delay_type min   -max_paths 10   -slack_lesser_than 999999  -to start_reg/D
  report_timing   -delay_type max   -max_paths 10   -slack_lesser_than 999999  -to stop_reg/D
  report_timing   -delay_type min   -max_paths 10   -slack_lesser_than 999999  -to stop_reg/D
  report_timing   -delay_type max   -max_paths 10   -slack_lesser_than 999999  -to sr_byte_reg[0]/D
  report_timing   -delay_type min   -max_paths 10   -slack_lesser_than 999999  -to sr_byte_reg[0]/D
  report_timing   -delay_type max   -max_paths 10   -slack_lesser_than 999999  -to sda_out_reg/D
  report_timing   -delay_type min   -max_paths 10   -slack_lesser_than 999999  -to sda_out_reg/D
  report_timing -delay_type max
  report_timing -delay_type min

