Allgemeines
============
Terminologie
------------
===========  ==========================================================================
Begriff      Bedeutung                                                                 
===========  ==========================================================================
Node         Freifunk-Router vor Ort                                                   
Gateway      Node mit der Fähigkeit Traffic ans Internet auszuleiten
Client       Rechner eines Nutzers, der mit einem Node verbunden ist.  
===========  ==========================================================================


besondere Adressen/Netze
------------------------
.. csv-table::
 :header-rows: 1
 :delim: ;
 
 Adresse; Bedeutung
 2a06:8187:fbab:2::1; next-node IP-Adresse: Diese Adresse ist jedem Node zugewiesen. Der Node ist für alle direkt verbundenen Clients unter dieser Adresse erreichbar.
 2a06:8187:fbab:1::/64;   eine bestimmte IP-Adresse in diesem Netz ist die local-node-IP-Adresse. Diese wird anhand der MAC-Adresse des Nodes bestimmt und lo zugewiesen.
 2a06:8187:fbab:1::/64;   Infrastruktur-Netz: In diesem Netz liegen Nodes und Serverkomponenten
 2a06:8187:fbab:2::/64;   Client-Netz: In diesem Netz liegen Clients

Jeder Node und jeder Client ist somit über eine öffentliche IPv6-Adresse erreichbar.


Firmware
========

Netzwerke
---------
Mesh
~~~~
Dieses Netz enthält alle Interfaces, die zur Aufrechterhaltung des Mesh genutzt werden. In der Standardkonfiguration mesh0 und mesh-vpn. Diese werden anhand :literal:`proto = gluon_mesh` identifiziert.

Client
~~~~~~
Dieses Netz enthält alle Interfces über die Clients eine Verbindung zum Node herstellen können.

WAN
~~~
Dieses Netz ist das LAN des Betreibers des Nodes.




Pakete
-------
l3roamd
~~~~~~~
l3roamd ermittelt Host-Routen von Clients und übergibt diese über die
Routingtabelle 11 an Babel. Der L3roamd kommuniziert über einen Netzwerkport
Der Dienst muss auf allen Nodes im Netzwerk ausgeführt werden.

gluon-l3roamd
~~~~~~~~~~~~~
Dieses Paket enthält ein initscript für den l3roamd. Die Initialisierung ist bewusst in einem separaten Paket untergebracht. Sobald der l3roamd ebenfalls dynamisch per Socket konfiguriert werden kann, entfällt dieses Paket. Die Initialisierung erfolgt dann per setup.d-script in gluon-mesh-babel.

mmfd
~~~~
Das Paket enthält ein mmfd-binary. Der mmfd flutet Nachrichten ins Layer 3-Netz. 

babeld
~~~~~~
Das Babel Protokoll ist für die Verteilung und Optimierung von Routen über verschiedene Nodes zuständig.
Es wird eine Entwicklungsversion von Babel 1.8 eingesetzt, weil nur diese die
Möglichkeit bietet zur Laufzeit auf die Konfiguration (Ein Socket auf Port ::1 33123) Einfluss zu nehmen.

Babel liest auf allen Nodes die Routingtabellen 11 und 12. In Tabelle 12 können
Babel statische Routen übergeben werden. Tabelle 10 wird für das routing
genutzt, sofern die Quell-IP im Mesh- oder Clientprefix liegt.


gluon-mesh-babel
~~~~~~~~~~~~~~~~
Dieses Paket integriert die einzelnen Komponenten mmfd, babeld, die firewall. babeld und mmfd werden durch ein in /lib/gluon/core/mesh/setup.d liegendes netifd-Script gestartet, sobald sich Interfaces ändern. Ob ein Interface für das mesh und damit für die Dienste Babel, l3roamd, mmfd relevant sind, wird anhand des hinterlegten Protokolls :literal:`proto = gluon_mesh` erkannt.


.. csv-table:: Firewall
 :header-rows: 1
 :delim: ;
 :stub-columns: 1
 
 Zone;    WAN; Client; Mesh; l3roamd; mmfd
 Interfaces;      br-wan      ; br-client             ; mesh-vpn, meshX                    ; l3roam0          ; mmfd0
 Protokolle OUT;  fastd,dns   ; -- ; -- ; -- ; --  
 Protokolle IN;    ssh         ; ssh, dns, http, ntp    ; http, babel, l3roamd, ssh, ntp, dns; --               ; --
 Protokolle both;  dhcp, ICMPx ; ICMPx	                ; ICMPx, respondd ; --               ; --
 Policy IN;	  DROP	       ; DROP                   ; DROP  ; ACCEPT; DROP (erlaube nur traffic vom lokalen Node)
 Policy OUT;	  DROP	       ; ACCEPT                 ; ACCEPT ; ACCEPT;  ACCEPT
 Policy FORWARD;   DROP	       ; DROP, (erlaube Traffic von und nach mesh sowie von und nach client); DROP (erlaube forward von und nach client und von und nach mesh); DROP (erlaube von und nach client); DROP


Auf dem Gerät laufen zwei Instanzen von dnsmasq. Eine Instanz dient der Möglichkeit der Namensauflösung für fastd. Per Firewall-Rule wird dem fastd-User erlaubt auf diese dnsmasq-Instanz auf Port 54 zuzugreifen.
Die zweite dnsmasq-Instanz dient als dns-Cache für angeschlossene Clients. Hier wird der AAAA-Record "nextnode" in die nextnode-IP-Adresse aufgelöst.

gluon-client-bridge-babel
~~~~~~~~~~~~~~~~~~~~~~~~~
Die Interfaces an denen sich Clients verbinden können (die Wifi-AP-Interfaces und die LAN-Ports) werden durch dieses Paket mit einer bridge verbunden. Der Bridge wird die next-node IP-Adresse zugewiesen. Dieses Paket erzeugt die Bridge und weist die Interfaces zu.

ffffm-dns-cache
~~~~~~~~~~~~~~~
Nodes verteilen mittels Router Advertisement ein nutzbares Prefix im
Freifunk-Netz. Außerdem wird per rdnssd der aktuelle Node als DNS-Server. Der DNS-Cache wird wie folgt konfiguriert: 

.. code:: lua

 dns = {
 cacheentries = 5000, 
 servers = { '2a06:8187:fb00:53::53' , } , 
 internaldomain = 'ffffm',  
 },   

* cacheentries gibt an, wie viele Einträge der Cache enthalten soll. Ein Eintrag benötigt ca 90 Byte RAM. Der RAM wird beim Starten des Nodes zugewiesen. 
* servers enthält die Liste der Upstream-DNS-Server an welche die ankommenden Anfragen im Fall eines Cache-Miss weitergeleitet werden
* internaldomain ist die netzwerkinterne Domain, die genutzt werden kann um hostnamen im Netz automatisch zu bestimmen. Dieser Parameter wird ausschließlich in Netzen mit IPv4-Unterstützung genutzt.



