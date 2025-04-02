# Saitek-FIP
Saitek / Logitech FIP (Flight Instrument Panel) display driver for Linux

# Driver
This is an attempt to reverse engineer Saitek / Logitech FIP (Flight Instrument Panel) driver for Windows which is not a real driver but more a software that drive display directly by USB commands via BULK commands.
Saitek FIP is using few endpoints

# What I've achieve until now
1. Build first attempt of driver using libusb https://libusb.info/
2. Search for USB device 0x06a3:0xa2ae .
3. Open USB device.
4. Send initialization command to the device.
5. Send command which enables image receiving.
6. Send BMP image to device.
7. Device is receiving the image and it shows it correctly.

# What I found until now
DirectOutput (which doesn't have anything to do with DirectX) has some functions. 
If you extract Flight_Instrument_Panel_x64_Drivers_8.0.150.0.exe from https://support.logi.com/hc/en-ch/articles/360024848713--Downloads-Flight-Instrument-Panel under MSI you will find DirectOutput_x64_Release.msi . 
This installer will install DirectOutput service (DirectOutputService.exe) and a DLL + .h header: DirectOutput.dll, DirectOutput.h.

So this is how the driver under Windows is working:
1. DirectOutputService.exe is used to communicate via USB with the device.
This service is receiving requests from plugins or other softwares via DirectOutput.dll and is creating the specific USB packet and is sending to the device
As I'm writing this document what is doing DirectOutputService.exe when you plugin the FIP:
  a. Searching for any USB device 0x06a3:0xa2ae.
  b. Sends initialization command to EndPoint 0x02: 00000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000
  c. Reads initialization done from device by reading data from Endpoint 0x81.
  d. Device must reply on Endpoint 0x81: 00000000000000000000000001000000000000000000000a0000000000000000000000000000000200000000
  e. DirectOutputService.exe is sending to device on endpoint 0x02 image receiving: 0000000000000001000384000000000000000000000000060000000000000000000000000000000000000000
  d. DirectOutputService.exe is sending a BMP RGB 8bit image to device in chunks of 512 bytes. The BMP image is just the RAW image without header.

2. DLL has these functions defined:
```
// HRESULT DirectOutput_Initialize(const wchar_t* wszPluginName);
// HRESULT DirectOutput_Deinitialize();
// HRESULT DirectOutput_RegisterDeviceCallback(Pfn_DirectOutput_DeviceChange pfnCb, void* pCtxt);
// HRESULT DirectOutput_Enumerate();
// HRESULT DirectOutput_RegisterPageCallback(void* hDevice, Pfn_DirectOutput_PageChange pfnCb, void* pCtxt);
// HRESULT DirectOutput_RegisterSoftButtonCallback(void* hDevice, Pfn_DirectOutput_SoftButtonChange pfnCb, void* pCtxt);
// HRESULT DirectOutput_GetDeviceType(void* hDevice, LPGUID pGuid);
// HRESULT DirectOutput_GetDeviceInstance(void* hDevice, LPGUID pGuid);
// HRESULT DirectOutput_SetProfile(void* hDevice, DWORD cchProfile, const wchar_t* wszProfile);
// HRESULT DirectOutput_AddPage(void* hDevice, DWORD dwPage, DWORD dwFlags)
// HRESULT DirectOutput_RemovePage(void* hDevice, DWORD dwPage)
// HRESULT DirectOutput_SetLed(void* hDevice, DWORD dwPage, DWORD dwIndex, DWORD dwValue);
// HRESULT DirectOutput_SetString(void* hDevice, DWORD dwPage, DWORD dwIndex, DWORD cchValue, const wchar_t* wszValue);
// HRESULT DirectOutput_SetImage(void* hDevice, DWORD dwPage, DWORD dwIndex, DWORD cbValue, const void* pvValue);
// HRESULT DirectOutput_SetImageFromFile(void* hDevice, DWORD dwPage, DWORD dwIndex, DWORD cchFilename, const wchar_t* wszFilename);
// HRESULT DirectOutput_StartServer(void* hDevice, DWORD cchFilename, const wchar_t* wszFilename, LPDWORD pdwServerId, PSRequestStatus psStatus);
// HRESULT DirectOutput_CloseServer(void* hDevice, DWORD dwServerId, PSRequestStatus psStatus);
// HRESULT DirectOutput_SendServerMsg(void* hDevice, DWORD dwServerId, DWORD dwRequest, DWORD dwPage, DWORD cbIn, const void* pvIn, DWORD cbOut, void* pvOut, PSRequestStatus psStatus);
// HRESULT DirectOutput_SendServerFile(void* hDevice, DWORD dwServerId, DWORD dwPage, DWORD cbInHdr, const void* pvInHdr, DWORD cchFile, const wchar_t* wszFile, DWORD cbOut, void* pvOut, PSRequestStatus psStatus);
// HRESULT DirectOutput_SaveFile(void* hDevice, DWORD dwPage, DWORD dwFile, DWORD cchFilename, const wchar_t* wszFilename, PSRequestStatus psStatus);
// HRESULT DirectOutput_DisplayFile(void* hDevice, DWORD dwPage, DWORD dwIndex, DWORD dwFile, PSRequestStatus psStatus);
// HRESULT DirectOutput_DeleteFile(void* hDevice, DWORD dwPage, DWORD dwFile, PSRequestStatus psStatus);
// HRESULT DirectOutput_GetSerialNumber(void* hDevice, wchar_t* pszSerialNumber, DWORD dwSize);
```
3. Any plugin for simulators is computing an image and is using functions from DirectOutput.dll, like DirectOutput_SetImage() to send image to the device. The data  sending data via DirectOutputService.exe which is communicats directly with the USB.

# What to do
If we want to write a driver we need to write also a library which communicates with the driver and to write plugins for simulators like X-Plane 12 for Linux or even MacOS.
This driver must have all functions found on DirectOutputService.exe .
Looking at the functions in DirectOutput.dll, I need to reverse engineering device AddPage, RemovePage, SetString, SetLed and also other functions.
From documentation found in DirectOutput: 
```
Image ID: 0
Size: 320 x 240 pixels
Description: Display an image on the FIP, data passed must be 24bpp RGB. The buffer size must be 320*240*3 = 230400 bytes. It is common to use the Bitmap image format for the FIP becuase the image data section of a 24bpp RGB Bipmap file conforms to the FIP buffer requirements. See Set_Image and Set_ImageFromFile for details on how to send images to the FIP.
```

From help Seems display supports multiple pages. I'm not sure if is integrated in the display itself or is a feature from DirectOutput.dll / DirectOutputService.exe .
