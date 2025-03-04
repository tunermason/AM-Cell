# LTE Base Station Setup for Amateur Radio Testing

> ⚠️ **WARNING** ⚠️
> 
> This setup is for educational and experimental purposes only. You must ensure compliance with local regulations and avoid interference with commercial ISPs. In the US, you may use CBRS bands; in other areas, you may use amateur bands under corresponding regulations. If using amateur bands, follow HamWAN's guidelines: [Internet and Part 97](https://hamwan.org/Administrative/Internet%20and%20Part%2097.html). Remember that amateur radio prohibits obscured communications (encryption), communications with pecuniary interest, and regular communications that could be provided by other radio services.

This guide will help HAMers set up a small LTE network for indoor, low-power testing purposes.

## Overview of LTE Network Architecture

An LTE network consists of three main components:

- User Equipment (UE): The end-user device, typically a smartphone or tablet.
- eNodeB: The base station that provides the radio interface between UEs and the core network.
- Evolved Packet Core (EPC): The core network that manages subscriber information, data routing, and connections to external networks.

```
  UE           eNodeB             EPC           External
(Phone)      (Base Station)   (Core Network)    Networks
  |               |                |                |
  |<--- Radio --->|<--- S1/X2 --->|<--- SGi ------>|
  |    Link       |    Interface   |   Interface    |
```

## Required Equipment

- A computer with Ubuntu 22.04 (or similar Linux distribution)
- A commercial picocell or femtocell for eNodeB
- Programmable SIM cards (sysmoUSIM-SJS1 recommended)
- SIM card reader/programmer
- Smartphones for testing

## Setting Up the Core Network (EPC)

We'll use Open5GS as our EPC. For detailed installation, refer to the official documentation.

## Quick Installation Steps:

```base
# Install MongoDB
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt update
sudo apt install mongodb-org

# Install Open5GS
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
sudo apt install open5gs

# Install Web UI (optional but recommended)
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
sudo apt install nodejs -y
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

## Programming SIM Cards

To use your own SIM cards with your LTE network, you'll need to program them:

1. Install SIM card programming tools:

```
sudo apt-get install pcscd pcsc-tools libccid libpcsclite-dev python3-pyscard
```
2. Get PySIM:
```
sudo apt-get install --no-install-recommends pcscd libpcsclite-dev python3 python3-setuptools python3-pyscard python3-pip
pip3 install --user -r requirements.txt
git clone git://git.osmocom.org/pysim
```
3. Program your SIM card:
```
./pySim-prog.py -p 0 -n "Amateur LTE" -a YOUR_ADM_KEY -s YOUR_ICCID -i YOUR_IMSI -x YOUR_MCC -y YOUR_MNC -k YOUR_KI -o YOUR_OPC
```
Ensure you replace placeholders with your own values.

## Setting Up Your Subscriber Data

1. Access the Open5GS WebUI at http://localhost:9999 (Username: admin, Password: 1423)
2. Navigate to the Subscriber section and add a new subscriber
3. Enter the IMSI, security context (K, OPC, AMF), and APN matching your SIM card

## Configuring the eNodeB (Commercial Picocell/Femtocell)

While specific instructions vary by manufacturer, most commercial picocells require:

1. Configuring the connection to the EPC (Open5GS) with the MME IP address
2. Setting the correct PLMN ID (MCC/MNC matching your SIM cards)
3. Configuring the frequency band (ensure it's within amateur radio allocations)
4. Setting the TAC (Tracking Area Code) to match your Open5GS configuration
5. Configuring the cell ID and physical cell identity (PCI)

Most commercial units provide a web interface for configuration.

### Configuring Open5GS MME for your eNodeB

Edit /etc/open5gs/mme.yaml to set the S1AP IP address, PLMN ID, and TAC:

```
mme:
  s1ap:
    server:
      - address: YOUR_MME_IP_ADDRESS
  gummei:
    plmn_id:
      mcc: YOUR_MCC
      mnc: YOUR_MNC
    mme_gid: 2
    mme_code: 1
  tai:
    plmn_id:
      mcc: YOUR_MCC
      mnc: YOUR_MNC
    tac: YOUR_TAC_VALUE
```
After changing, restart the MME service:

```bash
sudo systemctl restart open5gs-mmed.service
```

## Network Configuration

To allow your UEs to access external networks:

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Add NAT rule
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:cafe::/48 ! -o ogstun -j MASQUERADE

# Ensure firewall is configured properly
sudo ufw status
sudo ufw disable  # If active and causing issues
```

## SDR-Based Alternative

If you prefer an SDR-based eNodeB instead of commercial picocells, you can use srsRAN with a USRP B200/B210. For details, see the [Open5GS srsRAN guide](https://open5gs.org/open5gs/docs/tutorial/01-your-first-lte/).

## Testing Your Network

1. Power on your eNodeB and ensure it's connected to the EPC
2. Insert your programmed SIM into a test phone
3. Configure the phone's APN settings if needed
4. The phone should register on your network
5. Test with data traffic, calls between devices on your network, etc.

## Configuring APN on User Devices

For your devices to properly connect to your network, you'll need to configure the Access Point Name (APN) settings:

1. On your Android device:
   - Navigate to your phone's cellular/mobile network settings (location varies by manufacturer)
   - Add a new APN with the following details:
     - Name: Amateur LTE
     - APN: internet (this is the default APN in Open5GS)
     - Save the APN settings

2. On your iOS device:
   - Go to Settings > Cellular > choose the SIM you programmed > Cellular Data Network
   - Under "Cellular Data", configure:
     - APN: internet
     - Username: (leave blank unless specified in your Open5GS configuration)
     - Password: (leave blank unless specified in your Open5GS configuration)

Note: The default APN name in Open5GS is "internet" as specified in the [official configuration](https://github.com/open5gs/open5gs/blob/4012f572ed4602ce17a71161e39ea1051bae1f78/configs/open5gs/mme.yaml.in#L99). If you've customized your Open5GS configuration with a different APN name, use that instead.

## Monitoring

1. View logs in /var/log/open5gs/*.log
2. Use the Open5GS WebUI to monitor connected subscribers
3. Use Wireshark to analyze traffic if needed

## Important Notes

- Ensure all testing complies with amateur radio regulations
- Keep transmit power to the minimum necessary
- Use frequencies only within your permitted amateur radio bands
- This setup is for educational purposes only

For more detailed information, refer to the [Open5GS documentation](https://open5gs.org/open5gs/docs/).