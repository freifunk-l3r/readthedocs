Komponenten des Netzes
======================
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


fastd
-----
Nodes, die keine direkte WLAN-Verbindung haben, können über die VPN-Software fastd Teil eines gemeinsamen Netzes werden. In einem Batman-Netz werden die IP-Pakete des Freifunk-Netzes in Batman-Frames eingepackt und diese in fastd-Pakete gesteckt. Im Babel-Netz haben wir eine Verschachtelungsebene weniger, die IP-Pakete aus dem Freifunknetz können direkt auf den fastd-Interfaces transportiert werden. Die Datenströme zur Verwaltung des Netzes liegen daneben.

.. image:: MTU-calculation-helper-sheet-for-Freifunk-Networks.png
 :name: MTU in einem Batman-Netz

.. image:: MTU-calculation-helper-sheet-for-Freifunk-Networks_babel.png
 :name: MTU in einem Babel-Netz

Die MTU von fastd berechnet den im fastd-Paket liegenden Ethernet-Header nicht mit ein. Bei einer fastd-MTU von 1280 ist damit bereits fragmentierungsfreie IPv6-Kommunikation im Netz und außerhalb möglich.

Der Einsatz anderer VPN-Technologien (sogar im Parallelbetrieb zu fastd) ist denkbar. In Frankfurt haben wir uns entschieden nicht zu viele Komponenten gleichzeitig auszutauschen. Es gibt schnellere Lösungen als fastd, fastd hat sich jedoch bewährt.

babeld
------
babeld liest seinen Input für die Optimierung und Verteilung der Routen aus den Tabellen 11 (dorthin schreibt l3roamd) und 12. Der Rest funktioniert automatisch. Dabei ist folgende Konfiguration ein guter Startwert:
TODO Konfiguration einfügen

l3roamd
-------
l3roamd läuft auf jedem Node, damit clientrouten ermittelt und übermittelt werden können. Weil gegenwärtig l3roamd nicht ohne client-Interface läuft, wird dieses mit "lo" angegeben. Momentan muss das Kernelmodul CFG80211 geladen sein, damit l3roamd startet.

mmfd
----
mmfd kann genutzt werden um multicast (gegenwärtig broadcast) in einem l3-meshnetz umzusetzen. Benötigt wird das um respondd auf den nodes anzustoßen, Daten für die Map und das Monitoring bereitzustellen.

prefixd
-------
Dieser Dienst existiert noch nicht. TODO sobald sich das ändert, dokumentieren

respondd
--------
Der Dienst wird wie im Batman-Netz auch genutzt um Monitoring und Mapdaten im Netz zu transportieren.


