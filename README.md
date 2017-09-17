# STM32F_Uart_Bootloader

/*
	Firmare hex file format as follows
	: 02 0000 04 0801 F2
	1st byte ':' is line terminator which notify new line
	2nd byte '02' is datalength which notify datalength in this packet
	3rd ~ 4th '0000' is writed address.
	5th byte '04' is record type
		00 - Data commnad:Contains data and a 16-bit starting address for the data. 
		01 - End Of File: Must occur exactly once per file in the last line of the file. example: :00000001FF
		02 - Extended Segment Address: The data field contains a 16-bit segment base address(thus byte count is 02) 
			I think it is not used within STM32/PIC.
		04 - Extended Linear Address : Allows for 32 bit addressing (up to 4GiB). The address field is ignored (typically 0000) and the byte count is always 02. in packet ": 02 0000 04 0801 F2", it notifies that the Write address is 08010000
		05 - Start Linear Address: The address field is 0000 (not used) and the byte count is 04. 
			I think it is not used within STM32/PIC.
	6th ~ Last - 1th is data array
	Last byte F2: Checksum = two hex digits, a computed value that can be used to verify the record has no errors. 
	
	
	Likewise in ":10137000000000201400002044130E00FFFFFFFFB8" packet
		: 10 1370 00 00000020 14000020 44130E00 FFFFFFFF B8
	10- datalength is 16 bytes
	1370- Write address is 0x80011370
	00 - data packet
	00000020 14000020 44130E00 FFFFFFFF : 4 Long type data
	B8 - Checksum as is complement of sum of all bytes. 
*/


// Macros- defines section index in Packet as hex format.
#define BYTECOUNT 0
#define ADDRESS 1
#define RECORDTYPE 3
#define DATAPOINT 4

static unsigned long nSectorAddr = 0x08000000; // First start to write.

/*
	form *.hex file Firmware code line string :
	:020000040800F2 next
	:10000000FCFF012001000E00210F0E00210F0E0049
*/
unsigned char Hex_Line_Buffer[50]; // 


/*

	Because data packet is string (example 02 - ASCII format, in binary 0x3032),
	first is converted to binary.
	This routine acts in this way.
	
	parameter:
		unsigned char *pLine - packet array pointer	
		unsigned char nCnt - packet byte count
	note input value : except terminator ':' byte, example "10000000FCFF012001000E00210F0E00210F0E0049"

	after processing, pLine, nCnt value is follows
		0x10000000FCFF012001000E00210F0E00210F0E0049, 21  

*/

unsigned char Hex2Chr(unsigned char *pLine, unsigned char nCnt)
{
// convert for 2 byte ascii to one byte binary. 
    unsigned char index = 0;
    for(index = 0; index < (nCnt - 1) / 2; index++)
    { // it is not used string lib, because to decrease the size of bootloader,
	so that do not use memcpy, xtoi instruction.
	pLine[index] = *(pLine + index * 2) - 0x30;
 	pLine[index] << = 4;
       	pLine[index] += *(pLine + index * 2 + 1) - 0x30;
    }
    return index;
}

// Checksum calculation routine
unsigned char CheckSum(unsigned char *pLine, unsigned char nCnt)
{ 
	// using converted binary array, perform checksum.
    unsigned char index = 0;
    unsigned char nCheckSum = 0;
    for(index = 0; index < nCnt; index++)
        nCheckSum += *(pLine + index);
    return nCheckSum;
}

// Get 1 line of firmware, and analyse it.
// return value : 0 - arrived at end of file or error. 1- analyse OK.
unsigned char Get_Line_of_New_Firmware(unsigned char *Hex_Line_Buffer, unsigned char nLineCnt)
{
    unsigned char nDataLen = 0;
    unsigned long nAddr = 0;
    unsigned char nRectype = 0;
    unsigned long lData;
    unsigned short index = 0;

    // converting to binary array. 
    nLineCnt = Hex2Chr(Hex_Line_Buffer, nLineCnt);

    // calculate checksum
    if(CheckSum(Hex_Line_Buffer + 1, nLineCnt - 1) != 0)
    {// if it is not 0, checksum error
        // Error exception 
    }
    else
    {// Check OK.
	// Line analyse
	// Macros is defeined as your need. example #define BYTECOUNT 0

        nDataLen = Hex_Line_Buffer[BYTECOUNT]; // in packet datalength

	// Data write address, 
        nAddr = Hex_Line_Buffer[ADDRESS] << 8 + Hex_Line_Buffer[ADDRESS + 1];
	
	// record type
        nRectype = Hex_Line_Buffer[RECORDTYPE];

	// Packet processing
        switch(nRectype)
        {
            case 0x00:
            { // data packet
                for(index = 0; index < nDataLen / 4; index++)
                {
			// get data for 4 bytes
                    lData = 0;
                    lData += Hex_Line_Buffer[DATAPOINT + index * 4 + 1];
                    lData <<= 8;
                    lData += Hex_Line_Buffer[DATAPOINT + index * 4 + 0];
                    lData <<= 8;
                    lData += Hex_Line_Buffer[DATAPOINT + index * 4 + 3];
                    lData <<= 8;
                    lData += Hex_Line_Buffer[DATAPOINT + index * 4 + 2];

                    // Word Write: 4byte write
                    FLASH_Write((nSectorAddr + nAddr) + index * 4, lData);
                }

            }
            break;

            case 0x01:
            {
                // End of file
                return 0;
            }
            break;

            case 0x02:
            { // Extended Segment Address not supported in STM/PIC

            }
            break;

            case 0x03:
            { // Start Segment Addres not supported in STM/PIC

            }
            break;

            case 0x04:
            { // Extended Linear Address calculate
		//example in 02 0000 04 080E E4
		nSectorAddr = Hex_Line_Buffer[DATAPOINT + 0];
		nSectorAddr <<= 8;
		nSectorAddr += Hex_Line_Buffer[DATAPOINT + 1];
		nSectorAddr <<= 16;
		// real sector Address is 0x080E0000
            }
            break;

            case 0x04:
            { // Extended Linear Address

            }
            break;

            case 0x05:
            { // Start Linear Address

            }
            break;
        }
    }
    return 1;
}

/*
	Note:
	All of above routine should be placed in last part address of firmawre.
	
*/
