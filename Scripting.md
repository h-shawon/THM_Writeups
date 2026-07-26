# Task 1
Bash script
```bash
#!/bin/bash
#
# Decode a file that has been base64 encoded 50 times.
# Usage: ./decode50.sh [input_file]
#

set -e  # Exit on error

# Color codes for pretty output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Function to display usage
usage() {
    echo "Usage: $0 [input_file]"
    echo "  input_file: Path to the base64 encoded file (default: flag.txt)"
    exit 1
}

# Function to decode using base64 command
decode_with_base64() {
    local input_file="$1"
    local iterations="${2:-50}"
    local temp_file="temp_decoded.txt"
    
    echo -e "${BLUE}Starting decode process...${NC}"
    echo -e "File: $input_file"
    echo -e "Iterations: $iterations"
    echo ""
    
    # Copy input to temp file
    cp "$input_file" "$temp_file"
    
    # Decode in a loop
    for i in $(seq 1 $iterations); do
        # Decode the temp file and save to new temp
        base64 -d "$temp_file" > "${temp_file}.new" 2>/dev/null
        
        # Check if decode was successful
        if [ $? -ne 0 ]; then
            echo -e "${RED}❌ Error decoding at iteration $i${NC}"
            echo -e "${YELLOW}Partial content:${NC}"
            head -c 200 "$temp_file"
            echo "..."
            rm -f "$temp_file" "${temp_file}.new"
            return 1
        fi
        
        # Replace old temp with new
        mv "${temp_file}.new" "$temp_file"
        
        # Show progress (every 10 iterations)
        if [ $((i % 10)) -eq 0 ]; then
            echo -e "${GREEN}✓${NC} Decoded $i times..."
        fi
    done
    
    echo -e "\n${GREEN}✅ Successfully decoded $iterations times!${NC}"
    
    # Display the result
    echo -e "\n${BLUE}${SEPARATOR}${NC}"
    echo -e "${BLUE}FINAL FLAG:${NC}"
    echo -e "${BLUE}${SEPARATOR}${NC}"
    cat "$temp_file"
    echo -e "${BLUE}${SEPARATOR}${NC}"
    
    # Cleanup
    rm -f "$temp_file"
}

# Function to decode using while loop (alternative method)
decode_with_while_loop() {
    local input_file="$1"
    local iterations="${2:-50}"
    local temp_file="temp_decoded.txt"
    local counter=0
    
    echo -e "${YELLOW}Using while loop method...${NC}"
    
    cp "$input_file" "$temp_file"
    
    while [ $counter -lt $iterations ]; do
        base64 -d "$temp_file" > "${temp_file}.new" 2>/dev/null
        
        if [ $? -ne 0 ]; then
            echo -e "${RED}Error at iteration $((counter+1))${NC}"
            rm -f "$temp_file" "${temp_file}.new"
            return 1
        fi
        
        mv "${temp_file}.new" "$temp_file"
        counter=$((counter + 1))
        
        # Show progress
        if [ $((counter % 10)) -eq 0 ]; then
            echo -e "${GREEN}✓${NC} Decoded $counter times..."
        fi
    done
    
    echo -e "\n${GREEN}✅ Completed $counter iterations!${NC}"
    echo -e "\n${BLUE}${SEPARATOR}${NC}"
    echo -e "${BLUE}FINAL RESULT:${NC}"
    echo -e "${BLUE}${SEPARATOR}${NC}"
    cat "$temp_file"
    echo -e "${BLUE}${SEPARATOR}${NC}"
    
    rm -f "$temp_file"
}

# Main script
SEPARATOR="═══════════════════════════════════════════════════════════"

echo -e "${BLUE}${SEPARATOR}${NC}"
echo -e "${BLUE}  BASE64 DECODER (50 iterations)${NC}"
echo -e "${BLUE}${SEPARATOR}${NC}"

# Parse arguments
if [ $# -eq 0 ]; then
    INPUT_FILE="flag.txt"
    echo -e "${YELLOW}No file specified. Using default: $INPUT_FILE${NC}"
elif [ $# -eq 1 ]; then
    INPUT_FILE="$1"
else
    usage
fi

# Check if input file exists
if [ ! -f "$INPUT_FILE" ]; then
    echo -e "${RED}❌ Error: File '$INPUT_FILE' not found!${NC}"
    echo -e "Please create the file or specify the correct path."
    exit 1
fi

# Show file info
echo -e "Input file: ${GREEN}$INPUT_FILE${NC}"
echo -e "File size: $(wc -c < "$INPUT_FILE" | tr -d ' ') bytes"
echo -e "First 100 chars: $(head -c 100 "$INPUT_FILE")..."
echo ""

# Run the decoder (choose method)
decode_with_base64 "$INPUT_FILE" 50

# Uncomment to test while loop method:
# decode_with_while_loop "$INPUT_FILE" 50
```
# Task 2
Python script 
```python3
import socket
import time
import re

# Hard-coded IP
target_ip = 'machine_ip'

# Declaring request snippet
request = "GET / HTTP/1.1\r\nHost: %s\r\n\r\n" % target_ip

port = 3010  # Starting port
value = 0.0  # Starting value (use float for precision)
waiting = False
step = 0

print(f'Target: {target_ip}')
print(f'Starting Port: {port}')
print(f'Initial Value: {value:.2f}')
print('=' * 70)
print()

print(f'📍 Step {step}: Port {port}')

while True:
    try:
        # establishing the connection
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(5)  # Add timeout to prevent hanging
        s.connect((target_ip, int(port)))
        
        # sending the request declared above
        s.send(request.encode())
        r = s.recv(1024)
        r = r.decode().split('\n')[-1].strip()  # splitting and grabbing the last part
        
        if r != '':  # if response is not empty 
            print(f'Response: {r}')

        # Check for STOP
        if "STOP" in r.upper():
            print('\n' + '=' * 70)
            print('SUCCESS! Reached STOP!')
            print('=' * 70)
            break

        # Parse the response - handle potential extra spaces
        parts = r.split()
        if len(parts) >= 3:
            operation = parts[0].lower()
            v = float(parts[1])
            port = parts[2]  # Keep as string for now
            
            # Store old value for display
            old_value = value
            
            # Perform operation
            if operation == 'add':
                value += v
                print(f'  ➕ {old_value:.2f} + {v:.2f} = {value:.2f}')
            elif operation == 'minus' or operation == 'subtract':
                value -= v
                print(f'  ➖ {old_value:.2f} - {v:.2f} = {value:.2f}')
            elif operation == 'multiply' or operation == 'times':
                value *= v
                print(f'  ✖️ {old_value:.2f} × {v:.2f} = {value:.2f}')
            elif operation == 'divide' or operation == 'div':
                if v != 0:
                    value /= v
                    print(f'  ➗ {old_value:.2f} ÷ {v:.2f} = {value:.2f}')
                else:
                    print(f'Division by zero! Keeping {value:.2f}')
            elif operation == 'mod' or operation == 'modulo':
                value %= v
                print(f'{old_value:.2f} % {v:.2f} = {value:.2f}')
            else:
                print(f'Unknown operation: {operation}')
            
            step += 1
            print('-' * 50)
            print(f'📍 Step {step}: Port {port}')
            waiting = False
        else:
            print(f'Unexpected response format: {r}')
            # Try to recover by moving to next port
            port = str(int(port) + 1)
            print(f'Moving to next port: {port}')
            
        s.close()

    except ConnectionRefusedError:
        if not waiting:
            print('Waiting for connection... (port may be cycling)')
            waiting = True
        time.sleep(0.5)  # Small delay before retry
        
    except socket.timeout:
        if not waiting:
            print('Connection timeout, retrying...')
            waiting = True
        time.sleep(0.5)
        
    except socket.error as e:
        print(f'Socket error: {e}')
        time.sleep(1)
        
    except ValueError as e:
        print(f'Parse error: {e} - Response was: {r}')
        # Try to recover
        if 'r' in locals():
            # Attempt to extract port number from response
            port_match = re.search(r'(\d{4,5})', r)
            if port_match:
                port = port_match.group(1)
                print(f'Extracted port: {port}')
            else:
                port = str(int(port) + 1)
                print(f'Moving to next port: {port}')
        time.sleep(0.5)
        
    except Exception as e:
        print(f'Unexpected error: {e}')
        time.sleep(1)

# Print final result
print('\n' + '=' * 70)
print('FINAL RESULTS')
print('=' * 70)
print(f'Total steps completed: {step}')
print(f'Final value (unrounded): {value}')
print(f'Final value (rounded to 2 decimals): {value:.2f}')
print('=' * 70)
```
# Task 3
Python script
```python3
from cryptography.hazmat.primitives.ciphers.modes import GCM
from cryptography.hazmat.primitives.ciphers.algorithms import AES
from cryptography.hazmat.primitives.ciphers import Cipher
from cryptography.hazmat.backends import default_backend
from cryptography.exceptions import InvalidTag
import socket 
import hashlib

target_ip = 'machine_ip'
port = 4000


client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

request1 = client.sendto('hello'.encode(), (target_ip, port))
response1 = client.recvfrom(4069)
print(f"Response from tryhackme udp server: {response1}")

print("\n")

request2 = client.sendto('ready'.encode(), (target_ip, port))
response2 = client.recvfrom(4069)
print(f"Second response from tryhackme udp server: {response2}")
print("\n")

words_in_response2 = response2[0].split()
KEY = words_in_response2[0].split(b':')[1]
IV = words_in_response2[1].split(b':')[1]

checksum_to_find = response2[0].split()[14]


print("\n---\nKey: {}\nIV: {}\nChecksum to find: {}\n---\n".format(KEY.decode(), IV.decode(), checksum_to_find.hex())) 
    decryptor = Cipher(AES(key), GCM(iv, tag), backend=default_backend()).decryptor()
    plaintext = decryptor.update(ciphertext)
    decryptor.finalize()
    return plaintext


while True:
    client.sendto('final'.encode(), (target_ip, port))
    encrypted_text = client.recvfrom(4069)
    print(f'Encrypted text: {encrypted_text[0].hex()}\n')
    
    client.sendto("final".encode(), (target_ip, port))
    tag = client.recvfrom(1024)
    print(f'TAG : {tag[0].hex()}')
    
    try:
        decrypted_text = decryption(encrypted_text[0], KEY, IV, tag[0])
    except InvalidTag:
        print("InvalidTag: authentication failed for this ciphertext/tag — trying next packet.\n")
        continue

    try:
        printable = decrypted_text.decode()
    except UnicodeDecodeError:
        printable = decrypted_text.decode(errors='replace')
    print("Decrypted (decoded if possible):", printable)
    
    current_hash = hashlib.sha256(decrypted_text).hexdigest()
    print("Current hash: ", current_hash)
    
    if(current_hash == checksum_to_find.hex()):
        print("\n===\nWoohoo!! u cracked it '{}'\n===\n".format(printable))
        
        break
```