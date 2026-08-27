set TSU_DAT_MIN  100
set SDA_DLYMAX_VS_SCL [expr {$SCL_PERIOD - $TSU_DAT_MIN}]

set_input_delay \
    -max $SDA_DLYMAX_VS_SCL \
    -clock SCL_CLK \
    [get_ports sda_in]

set_input_delay \
    -min $THIGH_MIN \
    -clock SCL_CLK \
    [get_ports sda_in]


set TSU_START 600
set THD_START 600
set TSU_STOP  600
set THDBUFF  1300
