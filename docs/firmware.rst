Firmware
========

Netzwerke
---------
Mesh
~~~~
Dieses Netz enthält alle Interfaces, die zur Aufrechterhaltung des Mesh genutzt werden. In der Standardkonfiguration ist das das mesh0-Interface über welches Nodes mit WLAN-Kontakt ins Netz eingebunden werden und das mesh-vpn-Interface welches genutzt wird um Nodes per Internetverbindung zusammenzuschließen. Diese Interfaces werden anhand :literal:`proto = gluon_mesh` identifiziert.

Client
~~~~~~
Dieses Netz enthält alle Interfaces über die Clients eine Verbindung zum Node herstellen können.

WAN
~~~
Dieses Netz ist das LAN des Betreibers des Nodes.


Pakete
-------
l3roamd
~~~~~~~
l3roamd ermittelt Host-Routen von Clients und übergibt diese über die Routingtabelle 11 an Babel.
Der L3roamd kommuniziert über einen Netzwerkport.
Der Dienst muss auf allen Nodes im Netzwerk ausgeführt werden.

gluon-l3roamd
~~~~~~~~~~~~~
Dieses Paket enthält ein Initialisierungsscript für den l3roamd.
Die Initialisierung ist bewusst in einem separaten Paket untergebracht.
Sobald der l3roamd ebenfalls dynamisch per Socket konfiguriert werden kann, entfällt dieses Paket.
Die Initialisierung erfolgt dann per setup.d-script in gluon-mesh-babel.

mmfd
~~~~
Das Paket enthält ein mmfd-binary. Der mmfd flutet Nachrichten ins Layer 3-Netz. 

babeld
~~~~~~
Das Babel Protokoll ist für die Verteilung und Optimierung von Routen über verschiedene Nodes zuständig.
Im Zuge der gluon-babel Entwicklung wurde das Babel 1.8-Paket in openwrt integriert. So kann zur Laufzeit auf die Konfiguration von babeld über einen Socket auf ::1 mit Port 33123  Einfluss genommen werden. Zu diesem Socket baut der mmfd eine Verbindung auf um alle nodes im Netz zu ermitteln.

Babel liest auf allen Nodes die Routingtabellen 11 und 12.
In Tabelle 12 können Babel statische Routen übergeben werden.
Tabelle 10 wird für das Routing genutzt, sofern die Quell-IP im Mesh- oder Clientprefix liegt.


gluon-mesh-babel
~~~~~~~~~~~~~~~~
Dieses Paket integriert die einzelnen Komponenten mmfd, babeld, und die Firewall.
Babeld und mmfd werden durch procd (neu)gestartet, sobald sich Interfaces ändern.
Ob ein Interface für das mesh und damit für die Dienste Babel, l3roamd, mmfd relevant sind, wird anhand des hinterlegten Protokolls :literal:`proto = gluon_mesh` erkannt.


.. csv-table:: Firewall
 :header-rows: 1
 :delim: ;
 :stub-columns: 1
 
 Zone           ; WAN          ; Client                 ; Mesh                               ; l3roamd          ; mmfd
 Interfaces     ; br-wan       ; br-client              ; mesh-vpn, meshX                    ; l3roam0          ; mmfd0
 Protokolle OUT ; fastd, dns   ; --                     ; --                                 ; --               ; --
 Protokolle IN  ; ssh          ; ssh, dns, http, ntp    ; http, babel, l3roamd, ssh, ntp, dns,mmfd; --          ; respondd
 Protokolle both; dhcp, ICMPx  ; ICMPx                  ; ICMPx, respondd                    ; --               ; --
 Policy IN      ; DROP         ; DROP                   ; DROP                               ; ACCEPT           ; DROP (erlaube nur Traffic vom lokalen Node)
 Policy OUT     ; ACCEPT       ; ACCEPT                 ; ACCEPT                             ; ACCEPT           ; ACCEPT
 Policy FORWARD ; DROP         ; DROP, (erlaube Traffic von und nach mesh sowie von und nach client); DROP (erlaube forward von und nach client und von und nach mesh); DROP (erlaube von und nach client); DROP


Auf dem Gerät laufen zwei Instanzen von dnsmasq.
Eine Instanz dient der Möglichkeit der Namensauflösung für fastd.
Per Firewall-Rule wird dem fastd-User erlaubt auf diese dnsmasq-Instanz auf Port 54 zuzugreifen.
Die zweite dnsmasq-Instanz dient als dns-Cache für angeschlossene Clients.
Hier wird der AAAA-Record "nextnode" in die nextnode-IP-Adresse aufgelöst.

Interfaces, die das mesh-Protokoll ausführen, werden durch das gluon-core Paket automatisch der mesh-Firewall-Zone zugeordnet.

gluon-core
~~~~~~~~~~
Nodes verteilen mittels Router Advertisement ein nutzbares Prefix im Freifunk-Netz.
Außerdem wird per rdnssd der aktuelle Node als DNS-Server propagiert.
Der DNS-Cache wird wie folgt konfiguriert: 

.. code:: lua

 dns = {
 cacheentries = 4096, 
 servers = { '2001:DB8:3:53::53' , } , 
 },
 next_node = {
 name = 'nextnode',
 ...
 }

* cacheentries gibt an, wie viele Einträge der Cache enthalten soll. Ein Eintrag benötigt ca 90 Byte RAM. Der RAM wird beim Starten des Nodes zugewiesen. 
* servers enthält die Liste der Upstream-DNS-Server an welche die ankommenden Anfragen im Fall eines Cache-Miss weitergeleitet werden
* Der dnsmasq löst den unter next_node.name angegebenen Namen, hier "nextnode", in die next-node IP-Adresse auf.

Die Änderung ist bereits in gluon Verfügbar und auch in einem Batman-Netz nutzbar. In Frankfurt ist es bereits im Einsatz. Die DHCP-Server auf den Gateways müssen beim Einsatz in einem Batman-Netz so konfiguriert werden, dass diese als DNS-Server die nextnode IPv4-Adresse bekanntgeben.

