<img width="722" height="406" alt="image" src="https://github.com/user-attachments/assets/749c068a-27b7-4901-a706-e23e0aedb2d0" />


Lab design
- Three IOSv routers in a triangle
- Three autonomous systems: 65001, 65002, and 65003
- Directly connected IPv4 eBGP between R1 and R2
- Loopback-to-loopback IPv4 eBGP between R1 and R3
- Directly connected IPv6 eBGP between R2 and R3
- No annotations
- No preconfigured BGP, so students construct and troubleshoot every relationship
- Basic configurations include interfaces, loopbacks, IPv6 addressing, console settings, and only the host routes needed for the initial loopback exercise
