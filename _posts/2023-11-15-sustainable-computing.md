---
layout: post
tag: de
title: Nachhaltiges Computern
subtitle: Der Klimawandel in Deinem Kubernetes Cluster
date: 2023-11-14
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Nachhaltiges Computern
Nachhaltiges Computern, auch Green-IT genannt, ist die sorgsame Nutzung von Computer-Resourcen auf Hinblick der Umwelt. Zum einen sind das die Geräte selber und unter welchen Umständen deren Herstellung stattfindet. Aber auch der tägliche Gebrauch, die Höhe des Stromverbrauchs spielt hierbei eine Rolle, entsteht doch bei dessen Produktion mehr oder weniger CO2.

Im Zeitalter der Cloud steht der Computer irgendwo in einem Rechenzentrum. Mit einer virtuellen Maschine nutzt man nur einen Teil desselben und mit einem Pod in einem Kubernetes-Cluster wiederum nur ein winziges Teilstück davon. Aber davon unter Umständen viele.

# Strommix
Strom wird heutzutage sehr unterschiedlich produziert. Da ist nicht mehr nur ein Kraftwerk, welches durch Verbrennen von Kohle Turbinen antreibt, die dann letztlich einen Generator speisen, der dann den Strom liefert. Wir haben mittlerweile Windräder, Solarparks, Wasserkraftwerke, alle liefern Strom. Die Herstellung soll natürlich so billig wie möglich sein, also werden billigere Sorten bevorzugt. Der Bedarf ist zu unterschiedlichen Tageszeiten auch unterschiedlich gross.

Schauen wir uns den aktuellen Strommix in Deutschland an:

<img src="/images/2023-11-15_1.png"/>

Die Daten stammen von [Entso](https://transparency.entsoe.eu/), der transparenten Plattform zur Publizierung der Daten zur Generierung von Strom im pan-europäischen Raum.

# Emission
Emissionen sind die Summe aller Stoffe, die eine Energieerzeugungsanlage beim Betrieb ausstösst. Meist sind damit die Schadstoffe gemeint wie Schwefel, Teer, Rauch. Die Klimaforschung hat sich auf das Treibhausgas CO2 konzentriert. Für jede Energieerzeugungsart kann man jetzt Konstanten herleiten, wieviel Gramm CO2 pro kWh Strom abfallen. Bei Braunkohle sind das derzeit 966, bei Offshore Windparks 4. So kontant sind die Zahlen aber nicht. Die Technologie des Braunkohlekraftwerks entwickelt sich zum Beispiel ständig weiter und so kann man dessen Emission heutzutage nicht mit den Werken von 1950 oder 1990 vergleichen. Der Wirkungsgrad konnte beträchtlich gesteigert werden und es gibt kaum noch Abgase. Dennoch muss das Braunkohlekraftwerk als Stereotype für den verschleppte Klimaschutz herhalten.

In Summe kann man also in der Grafik oben einen Wert CO2g/s herleiten und daraus eine Energieampel erstellen, wann der Strom tatsächlich besonders grün ist.
Die Daten gelten deutschlandweit. In anderen Ländern kann man das in Regionen und Städten herunterbrechen, denn Strom geht immer den Weg des geringsten Widerstandes und so wird es regionale Nuancen geben, in welcher Region der Strom grüner ist. Solche Daten gibt es aber nicht für Deutschland in Echtzeit.

# Kepler
Um den Stromverbrauch elektrischer Geräte zu messen, bedarf es meist kleiner Messgeräte, die zwischen Steckdose und Gerät montiert werden. Daran kann man dann den durchfliesenden Strom und damit den Verbrauch messen. Bei Computern gibt es den [ACPI](https://de.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface), den Advanced Configuration and Power Interface. Da ist quasi das Messgerät schon im Computer und kann mit etlichen Tools abgefragt werden.
Komplizierter wurde das Thema bei Virtuellen Maschinen. Lange Zeit war es ziemlich wissenschaftlich, den Stromverbrauch von Virtuellen Maschinen zu messen. Das hat sich jetzt glücklicherweise geändert!
Und es geht noch einen Schritt weiter mit [Kepler](https://github.com/sustainable-computing-io/kepler) (Kubernetes-based Efficient Power Level Exporter). Ein Prometheus Exporter, der eBPF benutzt, um Messdaten von den CPUs und Linux Kernel Tracepoints zu bekommen. Dabei wird nicht nur cgroup-v2 sondern auch v1 unterstützt, funktioniert also auch noch mit älteren Systemen wie Ubuntu 20.04.

<img src="/images/2023-11-15_2.png"/>

Kepler bricht diese Metriken auf POD-Ebene im Kubernetes-Cluster herunter. Mit dem mitgelieferten Grafana-Dashboard kann man die Daten visuell darstellen.

# CO2 Fussabdruck
Bleiben wir bei der grafischen Darstellung. Im Grafana Dashboard `CaaS Carbon Footprint` bringen wir die Daten von Kepler und Entsoe zusammen. Kepler liefert ja schon mittels ServiceMonitor seine Daten für Prometheus, selbiges brauchen wir für Entsoe. Das Liefert [dieses Repo](https://github.com/caas-team/caas-carbon-footprint), das dortige Helm Chart bündelt die 2 Anwendungen und installiert sie in einen Kubernetes-Cluster.

Was kann man dann dort sehen? Weiter oben war schon mal ein Panel mit dem aktuellen Energiemix und einer Ampel, die anzeigt, ob es gerade ökologisch sinnvoll ist, seine Workload zu starten.

Wenn es erstmal läuft, kann man in diesem Panel die CO2 Gramm pro Stunde ablesen. In unserem Test-Cluster ist das freilig nicht so viel:

<img src="/images/2023-11-15_3.png"/>

# Fazit
Man wird so auf das Thema CO2 Fussabdruck aufmerksam gemacht und kann sich mit seiner Workload beschäftigen. Freilig wird die unter Umständen zu jeder Tages- und Nachtzeit laufen müssen. Oder man kann es mit Keda automatisch skalieren lassen, wie im letzten Beitrag gezeigt. Im [Repo](https://github.com/caas-team/caas-carbon-footprint/tree/main/examples) sind auch noch ein paar Beispiele, was man noch machen kann, um das Klima zu schützen.
