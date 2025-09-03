# Passi per Installare e configurare un Gateway su Raspberry Pi

## 1. Preparare l'Ambiente

1. **Aggiorna il Sistema Operativo:**
   Assicurati che il sistema operativo sia aggiornato.
   ```sh
   sudo apt update
   ```

2. **Installa le Dipendenze:**
   Installa `batctl` e abilita `batman-adv`.
   ```sh
   sudo apt install batctl -y
   sudo apt install iptables -y
   sudo apt install iptables-persistent -y
   sudo apt install isc-dhcp-server -y
   sudo apt install ifupdown -y
   ```
   Se durante l'installazione, ti verrà chiesto di salvare le regole correnti di `iptables`. Conferma selezionando "Yes".

## 2. Attiva `batman-adv`

1. **Carica il Modulo del Kernel `batman-adv`:**
   ```sh
   sudo modprobe batman-adv
   ```

2. **Rendi Permanente il Caricamento del Modulo:**
   Aggiungi `batman-adv` al file `/etc/modules` per caricarlo automaticamente all'avvio.
   ```sh
   echo "batman-adv" | sudo tee -a /etc/modules
   ```

## 3. Configurare l'interfaccia Ethernet e Wireless
Se hai il NetworkManager attivo, disattivalo.

### 1. Disattivare il Gestore di Rete
Se stai usando un gestore di rete come `NetworkManager`, devi disabilitarlo temporaneamente per configurare manualmente la tua rete mesh.
```sh
sudo systemctl stop NetworkManager
```

### 2. Disconnettere `wpa_supplicant`
`wpa_supplicant` potrebbe gestire l'interfaccia Wi-Fi. Fermalo prima di cambiare la modalità dell'interfaccia:
```sh
sudo systemctl stop wpa_supplicant
```

**Impedire al NetworkManager di gestire la rete wlan0:**
   Per impedire al Network manager di prendere il controllo di wlan0 con conseguente scomparsa della rete mesh bisogna editare il file di configurazione di ifupdown:
   ```sh
   sudo nano /etc/network/interfaces
   ```
   Aggiungi la linea (o modifica l'eventuale linea già dedicata a wlan0):
   ```sh
   iface wlan0 inet manual
   ```
   Salva il file e applica le modifiche.


**Impostare la modalità Ad-Hoc:**
```sh
sudo ip link set wlan0 down
sudo iwconfig wlan0 mode ad-hoc essid "mesh-network" ap any channel 1
sudo ip link set wlan0 up
```

 **Aggiungi l'Interfaccia Wi-Fi a `batman-adv`:**
```sh
sudo batctl if add wlan0
sudo ip link set up dev bat0
```

**Configura l'Indirizzo IP sull'Interfaccia `bat0`:**
Assegna un indirizzo IP statico all'interfaccia `bat0` nella stessa sottorete utilizzata dalle altre Raspberry Pi nella rete mesh.
```sh
sudo ifconfig bat0 10.0.0.1 netmask 255.255.255.0 up  # Modifica l'IP in base alla tua rete
```

## 3. Configurare la Condivisione della Connessione Internet

1. **Abilita il Forwarding IP:**
   Modifica il file `/etc/sysctl.conf` per abilitare il forwarding IP.
   ```sh
   sudo nano /etc/sysctl.conf
   ```
   Trova la linea:
   ```sh
   #net.ipv4.ip_forward=1
   ```
   Rimuovi il `#` per abilitare l'opzione:
   ```sh
   net.ipv4.ip_forward=1
   ```
   Salva il file e applica le modifiche:
   ```sh
   sudo sysctl -p
   ```

   In alternativa si può usare il comando:
   ```
   sudo sysctl -w net.ipv4.ip_forward=1
   ```

3. **Imposta le Regole di iptables per il NAT:**

### Passaggi per Installare `iptables`

1. **Verifica l'Installazione di iptables:**
   Verifica che `iptables` sia stato installato correttamente.
   ```sh
   iptables --version
   ```
2. Se il comando restituisce la versione di `iptables`, significa che è stato installato con successo.

3. **Crea la Directory per le Regole di `iptables`:**

   Crea manualmente la directory necessaria per salvare le regole di `iptables`:
   ```sh
   sudo mkdir -p /etc/iptables
   ```

4. **Salva le Regole di `iptables`:**

   Dopo aver creato la directory, salva le regole di `iptables` nel file `rules.v4`:
   ```sh
   sudo iptables-save | sudo tee /etc/iptables/rules.v4
   ```

5. Per fare in modo che le regole di `iptables` siano applicate automaticamente all'avvio, puoi usare `iptables-persistent`, un pacchetto che carica automaticamente le regole di `iptables` durante l'avvio del sistema.

6. **Verifica l'Installazione di `iptables-persistent`:**
   Puoi verificare che `iptables-persistent` sia configurato correttamente controllando lo stato del servizio:
   ```sh
   sudo systemctl status netfilter-persistent
   ```

7. **Ricarica le Regole di `iptables`:**
   Se necessario, ricarica le regole di `iptables` manualmente:
   ```sh
   sudo netfilter-persistent reload
   ```

8. Configura `iptables` per permettere la condivisione della connessione Internet tra l'interfaccia Ethernet (che ha accesso a Internet) e l'interfaccia mesh.
```sh
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
   Sostituisci eth0 con l'interfaccia di uscita verso internet, ad esempio wwan0
   
9. Configurare la Raspberry come Gateway
```sh
sudo batctl gw_mode server
```

## 5. Configurazione del server DHCP
```sh
sudo nano /etc/dhcp/dhcpd.conf
```
Inserire questa configurazione:
```
subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.10 10.0.0.100;
    option routers 10.0.0.1;  # L'indirizzo IP del gateway (bat0)
    option broadcast-address 10.0.0.255;
    option domain-name-servers 8.8.8.8, 8.8.4.4;  # DNS
}
```
```
sudo nano /etc/default/isc-dhcp-server
```
Inserire `bat0` nelle inferfacce:
```
INTERFACESv4="bat0"
```
Infine restartare il server dhcp:
```sh
sudo systemctl restart isc-dhcp-server
```
## 6. Verifica la Configurazione (prima configura una raspberry per il nodo mesh)

Per questa sezione potrebbe essere necessario farla in un secondo momento, quando saranno installate anche le raspberry della rete mesh.

1. **Controlla la Configurazione di `batman-adv`:**
   Usa `batctl` per assicurarti che l'interfaccia mesh sia correttamente configurata.
   ```sh
   sudo batctl n  # Mostra i vicini nella rete mesh
   sudo batctl o  # Mostra la tabella di routing
   ```

2. **Testa la Connettività alla Rete Mesh:**
   Verifica che la prima Raspberry Pi possa comunicare con le altre Raspberry Pi nella rete mesh.
   ```sh
   ping 10.0.0.2  # Sostituisci con l'indirizzo IP di un'altra Raspberry Pi
   ```

3. **Testa la Condivisione della Connessione Internet:**
   Controlla che le altre Raspberry Pi possano accedere a Internet tramite la prima Raspberry Pi.
   ```sh
   ping google.com
   ```
   

# Passaggi per configurare le Raspberry Pi come nodi della mesh

#### 1. Configurare Raspberry Pi

Per la seconda e terza Raspberry Pi, devi configurarle per partecipare alla rete mesh e accedere a Internet tramite la prima Raspberry Pi (gateway).

Ripeti i seguenti passaggi su entrambe le Raspberry Pi (seconda e terza):

1. **Installa `batctl`:**

   ```sh
   sudo apt update
   sudo apt install batctl -y
   ```

2. **Carica il Modulo del Kernel `batman-adv`:**

   ```sh
   sudo modprobe batman-adv
   ```

3. **Rendi Permanente il Caricamento del Modulo:**

   ```sh
   echo "batman-adv" | sudo tee -a /etc/modules
   ```

### 2. Disattivare il Gestore di Rete
Se stai usando un gestore di rete come `NetworkManager`, devi disabilitarlo temporaneamente per configurare manualmente la tua rete mesh.
```sh
sudo systemctl stop NetworkManager
```

### 3. Disconnettere `wpa_supplicant`
`wpa_supplicant` potrebbe gestire l'interfaccia Wi-Fi. Fermalo prima di cambiare la modalità dell'interfaccia:
```sh
sudo systemctl stop wpa_supplicant
```

### 4. Impostare la modalità Ad-Hoc:
```sh
sudo ip link set wlan0 down
sudo iwconfig wlan0 mode ad-hoc essid "mesh-network" ap any channel 1
sudo ip link set wlan0 up
```

### 5. Aggiungi l'Interfaccia Wi-Fi a `batman-adv`:
```sh
sudo batctl if add wlan0
sudo ip link set up dev bat0
```
### 6. Configurare la Raspberry come Gateway
```sh
sudo batctl gw_mode client
```

### 7. Chiama il DHCP Client per assegnazione IP
```sh
sudo dhclient bat0
```
### 8. Verificare la Configurazione della Rete Mesh

1. **Controlla lo Stato della Rete Mesh su Tutte le Raspberry Pi:**

   Utilizza `batctl` per verificare che le Raspberry Pi siano collegate correttamente alla rete mesh:

   ```sh
   sudo batctl n  # Mostra i vicini della rete mesh
   sudo batctl o  # Mostra la tabella di routing
   ```

   Dovresti vedere le altre Raspberry Pi come vicine nella rete mesh.

2. **Aggiungere gateway**
```
sudo route add default gw 10.0.0.1 #Sostituire con l`ip del gateway
```
3. **Testa la Connettività di Rete:**

   Esegui un ping da ciascuna Raspberry Pi alle altre per assicurarti che la connettività sia stabilita:

   - Dalla seconda Raspberry Pi:
     ```sh
     ping 10.0.0.1  # Ping alla prima Raspberry Pi (gateway)
     ping 10.0.0.3  # Ping a un`altra Raspberry Pi
     ```

4. **Testa l'Accesso a Internet:**

   Verifica che la seconda e la terza Raspberry Pi possano accedere a Internet tramite la prima Raspberry Pi:

   ```sh
   ping 8.8.8.8
   ```

# Impostazioni automatche al riavvio
Modificare il file `/etc/rc.local` e aggiungere prima di `exit 0`
1. Per Gateway
   ```
   sudo ip link set wlan0 down
   sudo iwconfig wlan0 mode ad-hoc essid "mesh-net" ap any channel 1
   sudo ip link set wlan0 up
   sudo batctl if add wlan0
   sudo ip link set up dev bat0
   sudo ifconfig bat0 10.0.0.1 netmask 255.255.255.0 up
   sudo batctl gw_mode server
   sudo systemctl restart isc-dhcp-server
   ```
   Se si vuole che sia attivo il client dhcp sulla rete eth0 bisogna aggiungere il comando:
   ```
   sudo dhclient eth0
   ```
   Se si usa wwan0 non serve in quanto è tutto gestito dal nmcli
   
3. Per Nodo Mesh
```
sudo ip link set wlan0 down
sudo iwconfig wlan0 mode ad-hoc essid "mesh-net" ap any channel 1
sudo ip link set wlan0 up
sudo batctl if add wlan0
sudo ip link set up dev bat0
sudo batctl gw_mode client
sudo dhclient bat0
sudo route add default gw 10.0.0.1
```
