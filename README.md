# Integrating physical devices with IOTA — The IOTA debit card, Part 1

The 6th part in a series of beginner tutorials on integrating physical devices with the IOTA protocol.

![img](https://miro.medium.com/max/700/1*zaz8OoM4tX1iOdfK6i-6xA.jpeg)

------

## Introduction

This is the 6th part in a series of beginner tutorials where we explore integrating physical devices with the IOTA protocol. This tutorial is the first part in a sequence of tutorials where we will try to replicate a traditional fiat based debit card payment solution with an IOTA based solution. In this first tutorial we will concentrate on the debit card itself as we learn how to read and write information to the card. In the second tutorial we will be integrating our new payment solution with a physical device that will allow us to pay for its service using the IOTA debit card. Finally we will take a look at implementing a PIN code protection mechanism as an additional level of security when using the IOTA debit card.

*Warning!*
*Before we continue with this tutorial i would like to issue a warning with respect to not using large amounts of IOTA tokens when playing around with these tutorials. There is always something that can go wrong, and when it does, make sure you don’t loose any extensive amount of IOTA’s. This warning is especially true for this tutorial as we will be creating and storing seed’s on an RFID tag. If something where to happen to the tag, you may not be able to recover the seed. One preventive measure you can take is to make a copy of every seed you save on to a tag in case you need to recover it later on.*

------

## The Use Case

Now that our hotel owner has his new cleaning log system in place as described in the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-using-rfid-with-iota-868c15e0a040) he wants to tackle an even greater problem that has been bugging him for a while. This problem is related to how he can charge his hotel guests for using common appliances and services located in his hotel; such as the coffee maker in the hotel reception or the swimming pool lockers. People don’t carry coins around anymore and implementing a Visa based payment system would simply be to complex and expensive. As an enthusiastic supporter of the IOTA technology he would like his new payment system to be based on the IOTA token. But to be honest, must people don’t own IOTA’s and have no idea how to get them. Also, most people don’t have an IOTA wallet installed on there mobile phones and let alone know how to use it. After scratching hos head over this problem for a while he comes up with the perfect solution.

What if the guest could use his hotel key card to pay the coffee maker or swimming pool locker? In that case, the guest would simply go to the reception and purchase any quantity of IOTA tokens that would be transferred to his key card. As such, effectively turning his key card in to an IOTA debit card. Each time the guest uses his IOTA debit card to pay for a service, the appropriate amount of tokens will subtracted from his key card and transferred back to the hotel owners IOTA account.

But wait a second, you can store IOTA’s on a key card?

Well, not rally, but you can store an IOTA seed that will allow you to access and spend IOTA’s related to that particular seed.

The main objective of this first tutorial is to implement the functionality required by the hotel owner to create and issue new IOTA debit cards for his guests. This will include functions such as:

- Creating and writing an IOTA seed to a key card
- Retrieving and displaying an IOTA seed from a key card
- Checking the balance for an IOTA seed stored on a key card
- Creating new and unused addresses for a seed stored on a key card in case he wants to send additional funds to the card.

This all sounds great but how do we accomplish this in practice? Well, short answer is, we will use our MFRC522 RFID reader/writer from the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-using-rfid-with-iota-868c15e0a040) to effectively convert an RFID tag to an IOTA debit card.

*Note!
You may ask yourself why there is no function to transfer funds to the card in this list? The simple answer is that it would require sending valued IOTA transactions using the PyOTA library, which is a topic we will save for the next tutorial. Until then, when the hotel owner wants to transfer IOTA tokens to a particular card, he simply uses his favorite IOTA wallet and transfer the funds to the address shown when using the “* **Display IOTA debit card seed transfer address***” function.*

------

## The Mifare RFID tag

In this section we will take a closer look at the RFID tag itself. Check out the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-using-rfid-with-iota-868c15e0a040) for more information about RFID in general, the MFRC522 RFID module and how to install and use it with a Raspberry PI.

Now that we want to read and write custom data to/from the tag, as appose to the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-using-rfid-with-iota-868c15e0a040) where we just retrieved the internal ID of the tag, we should take a closer look at the tag itself and how it works.

The RFID tags we are using in this tutorial are commonly known as the Mifare series of RFID tags. A common feature for all tag’s in the Mifare series is that they operates at 13.56MHz, and that they support authentication and data encryption. The Mifare tag’s typically comes in the shape of a key ring or credit card.

![img](https://miro.medium.com/max/640/1*eFgFh0ak6UdbXBzZbDrcxQ.jpeg)

Before going in details with respect to reading and writing data to/from the Mifare tag we should take a moment and discuss the topic of authentication and encryption. As mentioned above, the Mifare series includes an authentication and encryption mechanism that prevents data from being read or written to the tag without proper authentication. This is a nice feature that we will explore in a future tutorial. However, until then you need to be very careful not to overwrite the default authentication key by accident as this may render the tag useless. More about this in the next section.

------

## Reading and Writing data with the Mifare RFID tag

The Mifare RFID tag functions more or less like any other EEPROM where the memory is organized in 16 sectors of 4 blocks. Each block contains 16 bytes. The first data block (block 0) of the first sector (sector 0) contains the IC manufacturer data. This block is programmed and write protected during manufacturing of the tag. Notice that the last block in each sector is identified as “Key A | Access Bits | Key B”, this is where the authentication key for each sector is stored. To read or write information to a block in that sector you need to provide a valid authentication key for the sector. If you fail to provide a valid authentication key, you will get an authentication error. So if you ever overwrite the last block in a sector, make sure you remember or make a note of the data that was written to the block as this now functions as the authentication key for the sector.

*Note!*
*In this first tutorial we will only be using the default authentication key that was written to the tag when it was manufactured. For reference, this authentication key is as follows: [0xFF,0xFF,0xFF,0xFF,0xFF,0xFF].*

*Warning!*
*As we are not changing the default authentication key for the tag in this tutorial, it would be easy for any bad actor who has access to the card to just recreate the IOTA seed stored on the tag and steal its funds.*

![img](https://miro.medium.com/max/639/1*b-FazrBCLP1JfenZsgByhA.png)

**Writing to the tag**
When writing data to the Mifare tag we use the *MFRC522_Write* function from the MFRC522-python library. The *MFRC522_Write* function takes two arguments:

1. The ID of the block you want to write to.
2. The data you want to write to the block

Notice that the data must be in the form of a list of 16 bytes, where each byte is represented by its numeric ASCII value.

*Note!*
*The Python code used in my example will split the seed into 6 elements that will be written to block 8, 9, 10, 12, 13 and 14. Notice that block 11 is excluded as this block is used for the sector 2 authentication key.*

**Reading from the tag**
When reading data from the Mifare tag we use the *MFRC522_Read* function from the MFRC522-python library. The *MFRC522_Read* function takes one argument:

1. The ID of the block you want to read.

***The function returns a list of bytes where each byte is represented by its numeric ASCII value.

*Note!*
*Notice that there is no argument in either function specifying the sector, meaning that if you specify block 8 as the argument, it effectively means block 0 in sector 2 according the table shown above.*

***Important!*
*If you open the MFRC522.py file that is installed with the MFRC522-python library and look for the MFRC522_Read function you will notice that the function basically just prints the values retrieved from the tag and doesn’t return anything. This will not work for our project as we need these values for re-building our seed. More on how to fix this problem later on.*

------

## Software and libraries

If you followed [Gus’s tutorial](https://pimylifeup.com/raspberry-pi-rfid-rc522/) on installing the MFRC522 with the Raspberry PI from the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-using-rfid-with-iota-868c15e0a040) you would have installed the SimpleMFRC522 library for communicating with the MFRC522. The SimpleMFRC522 library is a simplified version (fork) of the standard MFRC522-python library, which is great for getting started with the MFRC522 without having to deal with the complexity of memory blocks and authentication.

In this project however, we need to interface the standard MFRC522-python library directly.

The MFRC522-python library can be downloaded from [here](https://github.com/mxgxw/MFRC522-python).

Or using..

git clone https://github.com/mxgxw/MFRC522-python.git

*Note!
If you installed the SimpleMFRC522 library as described in the previous tutorial it could be that the MFRC522-python library installation folder uses the same folder name as the* SimpleMFRC522 *installation folder. If this is the case then you might need to rename the original folder to something else before installing the MFRC522-python library.*

*Important!*
*When you are finished installing the MFRC522-python library there is one more thing we need to do to get everything working correctly. Remember we talked about the MFRC522_Read function not returning any data? Use the following steps to fix this issue:*

1. *Open the MFRC522.py file from the folder where you installed the MFRC522-python library.*
2. *Locate the MFRC522_Read function.*
3. *Add the following line at the end of the function:*
   **return backData**
4. *Save the file*

We also need to make sure the PyOTA library is installed if you haven’t installed it already. The PyOTA library with installation instructions can be found [here](https://github.com/iotaledger/iota.lib.py).

------

## The Python code

Before moving on to the actual Python code for this project, let’s do a quick recap of the functions included in the script.

When launching the script you will be presented with a menu that has the following options:

1. **Assign manual seed to IOTA debit card**
   This option will allow you write a pre-defined seed to the RFID tag. The function will ask you for a 81 character seed before validating the seed. Then you will be asked to hold the RFID tag close to the reader. As soon as the RFID reader detects the tag, the seed will be written to the tag.
2. **Assign automatic seed to IOTA debit card**
   Using this option the Python script will generate a random seed using the Python random function. You will then be asked to hold the RFID tag close to the reader. As soon as the RFID reader detects the tag, the seed will be written to the tag. You will get a confirmation showing the seed that was generated and written to the tag.
3. **Display seed stored on IOTA debit card**
   Using this option you can retrieve and display the seed currently stored on the card.
4. **Display IOTA debit card seed balance**
   Using this option you can check the amount of IOTA tokens currently available for the seed stored on the card.
5. **Display IOTA debit card seed transfer address**
   Using this option you will get the next available seed address in case you want to add additional funds to the seed. It is important that we don’t add any funds to an address that has already been spent from due to the IOTA one time signature protection. In this case we use the PyOTA get_new_addresses function with the count=None parameter to return the next available address that has no transaction history on the Tangle.

```python
from iota import Iota
import random
import RPi.GPIO as GPIO
import MFRC522
import signal
from textwrap import wrap

# Set seed to <blank>
seed=""

# URL to IOTA fullnode used when checking balance and free addresses
iotaNode = "https://nodes.thetangle.org:443"

# Function the generates and returns a random seed
def generate_seed():
    chars=u'9ABCDEFGHIJKLMNOPQRSTUVWXYZ'
    rndgenerator = random.SystemRandom()
    new_seed = u''.join(rndgenerator.choice(chars) for _ in range(81))
    return new_seed

# Function the validates seed length
def seed_check_length(seed):
    if len(seed) == 81:
        return True
    else:
        return False

# Function that validates seed characters
def seed_check_chars(seed):
    allowed_chars = set("ABCDEFGHIJKLMNOPQRSTUVWXYZ9")
    if set(seed).issubset(allowed_chars):
        return True
    else:
        return False

# Show main menu
print ("""
1.Assign manual seed to IOTA debit card 
2.Assign automatic seed to IOTA debit card
3.Display seed stored on IOTA debit card
4.Display IOTA debit card seed balance
5.Display IOTA debit card seed transfer address
""")

# Get user selection
ans=input("What would you like to do? ")

# In case 1, get manual seed from user
if ans==1:
    seed = raw_input("\nWrite or paste seed here: ")

    # Check that seed length is correct
    if seed_check_length(seed) == False:
        print("Invalid seed length")
        exit()

    # Check that seed contains only valid characters
    if seed_check_chars(seed) == False:
        print("Seed contains invalid characters")
        exit()

# In case 2, get seed from seed generator function
elif ans==2:
    print("\nGenerating new seed...")
    seed = generate_seed()
   
continue_reading = True
       
# Capture SIGINT for cleanup when the script is aborted
def end_read(signal,frame):
    global continue_reading
    print "Ctrl+C captured, ending read."
    continue_reading = False
    GPIO.cleanup()

# Function to write seed to IOTA debit card
def write_seed(seed):
    
    # Add additional characters to seed so that we have a total of 6x16 characters 
    seed = seed + '999999999999999'       

    # Convert seed to a list of 6x16 characters  
    mylist = wrap(seed, 16)

    # Write seed over 6 separate blocks
    write_block(8, mylist[0])
    write_block(9, mylist[1])
    write_block(10, mylist[2])           
    write_block(12, mylist[3])
    write_block(13, mylist[4])
    write_block(14, mylist[5])


# Function to write single block to RFID tag
def write_block(blockID, str_data):

    status = MIFAREReader.MFRC522_Auth(MIFAREReader.PICC_AUTHENT1A, blockID, key, uid)
    
    if status == MIFAREReader.MI_OK:
        
        data=[]
        for letter in str_data:
            data.append(ord(letter))
        MIFAREReader.MFRC522_Write(blockID, data)
        
    else:
        print "Authentication error"


# Function that reads the seed stored on the IOTA debit card
def read_seed():
    
    seed = ""
    seed = seed + read_block(8)
    seed = seed + read_block(9)
    seed = seed + read_block(10)
    seed = seed + read_block(12)
    seed = seed + read_block(13)
    seed = seed + read_block(14)
    
    # Return the first 81 characters of the retrieved seed
    return seed[0:81]

# Function to read single block from RFID tag
def read_block(blockID):

    status = MIFAREReader.MFRC522_Auth(MIFAREReader.PICC_AUTHENT1A, blockID, key, uid)
    
    if status == MIFAREReader.MI_OK:
        
        str_data = ""
        int_data=(MIFAREReader.MFRC522_Read(blockID))
        for number in int_data:
            str_data = str_data + chr(number)
        return str_data
        
    else:
        print "Authentication error"


# Hook the SIGINT
signal.signal(signal.SIGINT, end_read)

# Create an object of the class MFRC522
MIFAREReader = MFRC522.MFRC522()

# Display scan IOTA debit card message
print("\nHold IOTA debit card close to reader...")

# This loop keeps checking for near by RFID tags. If one is found it will get the UID and authenticate
while continue_reading:
       
    # Scan for cards    
    (status,TagType) = MIFAREReader.MFRC522_Request(MIFAREReader.PICC_REQIDL)

    # If a card is found
    if status == MIFAREReader.MI_OK:
        print "Card detected"
    
    # Get the UID of the card
    (status,uid) = MIFAREReader.MFRC522_Anticoll()

    # If we have the UID, continue
    if status == MIFAREReader.MI_OK:

        # Print UID
        print "Card read UID: %s,%s,%s,%s" % (uid[0], uid[1], uid[2], uid[3])
    
        # This is the default key for authentication
        key = [0xFF,0xFF,0xFF,0xFF,0xFF,0xFF]
        
        # Select the scanned tag
        MIFAREReader.MFRC522_SelectTag(uid)
        
        # Write seed to IOTA debit card
        if ans==1 or ans==2:
            
            # Write seed to IOTA debit card
            write_seed(seed)
            
            # Show end message
            print("\nThe following seed was sucessfully added to the IOTA debit card")
            print(seed[0:81])
            
        
        # Display seed stored on IOTA debit card    
        elif ans==3:
            
            # Get seed from IOTA debit card
            seed=read_seed()
            
            # Display seed stored on the IOTA debit card
            print("\nThe IOTA debit card has the following seed: ")
            print(seed)
        
        # Display balance of seed stored on IOTA debit card
        elif ans==4:
            
            # Get seed from IOTA debit card
            seed=read_seed()

            # Create an IOTA object
            api = Iota(iotaNode, seed)
            
            print("\nChecking IOTA debit card balance. This may take some time...")

            # Get balance for the IOTA debit card seed
            card_balance = api.get_account_data(start=0, stop=None)
            
            balance = card_balance['balance']
            
            print("\nIOTA debit card balance is: " + str(balance) + " IOTA")
        
        # Display next unused address of seed stored on IOTA debit card
        elif ans==5:
            
            # Get seed from IOTA debit card
            seed=read_seed()
            
            # Create an IOTA object
            api = Iota(iotaNode, seed)
            
            print("\nGetting next available IOTA address. This may take some time...")
            
            # Get next available address without any transactions
            result = api.get_new_addresses(index=0, count=None, security_level=2)
            
            addresses = result['addresses']
            
            addr=str(addresses[0].with_valid_checksum())
            
            print("\nUse the following IOTA address when transfering new funds to IOTA debit card:")
            print(addr)

           
        # Stop reading/writing to RFID tag
        MIFAREReader.MFRC522_StopCrypto1()
        
       
        # Make sure to stop reading for cards
        continue_reading = False
```

You can download the source code from [here](https://gist.github.com/huggre/1a317747d7686acd3e71d0dad761a459)

------

## Running the project

To run the the project, you first need to save the code in the previous section as a text file in the same folder as where you installed the MFRC522-python library.

Notice that Python program files uses the .py extension, so let’s save the file as **iota_debit_card.py** on the Raspberry PI.

To execute the program, simply start a new terminal window, navigate to the folder where you saved *iota_debit_card.py* and type:

**python iota_debit_card.py**

You should now see the code being executed in your terminal window displaying the main menu.

------

## What's next?

Our IOTA debit card would not be very practical unless there was some way of using it. So, in the next tutorial we will integrate our new payment solution with a physical device that will allow us to pay for its services using the IOTA debit card. Stay tuned…

------

## Donations

If you like this tutorial and want me to continue making others, feel free to make a small donation to the IOTA address shown below.

![img](https://miro.medium.com/max/400/1*kV_WUaltF4tbRRyqcz0DaA.png)

> GTZUHQSPRAQCTSQBZEEMLZPQUPAA9LPLGWCKFNEVKBINXEXZRACVKKKCYPWPKH9AWLGJHPLOZZOYTALAWOVSIJIYVZ
