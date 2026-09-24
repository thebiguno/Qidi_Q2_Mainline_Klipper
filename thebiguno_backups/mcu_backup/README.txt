These are the backups of the mcu board (q2-x9-3-v1.3-stock.bin) and toolhead board (q2-a10-v1.1.3-stock.bin) running the stock Qidi Q2 firmware verison 1.1.1.

It was read via STLink with the command:

st-flash --connect-under-reset read q2-x9-3-v1.3-stock.bin 0x08000000 0x80000
st-flash read q2-a10-v1.1.3-stock.bin 0x08000000 0x20000 
