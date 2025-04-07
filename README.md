# waveboard
Repository for the control software of the waveboard

## Installation
add 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/user/Waveboard/lib' to your '.bashrc' file


## Acquisizione senza controller

Entrare in SSH nella scheda e eseguire i sefuenti comandi: 

Imposta la data (nell'esempio 7 aprile 2025 ore 15:01)
``` bash
date 0407150125
```

Imposta il gain alto (20x). Usare il comando "\$gsla#" per il gain basso (2x)
``` bash
./M4Comm -s "\$gsha#" &
```

Importa i registri del tempo (roba necessaria per far funzionare la scheda)
``` bash
./SetTimeReg -t l
```

Imposta gli IOB delay (altra roba che serve a far funzionare la scheda correttamente)
``` bash
bash daq_set_iob_delay.sh -N "{0..11}" -F "25 25 20 20 18 18 18 18 25 25 20 20"
```

Imposta i piedistalli (esempio con i valori della WB 1n):
```bash 
bash daq_set_pedestal.sh -N "0 1 2 3 4 5 6 7 8 9 10 11" -P " 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0 0x03DC0"
```
Questi sono valori alti (usati per i segnali negativi). Se volete acquisire segnali positivi servono valori di piedistallo bassi "350 330 325 338 345 322 0x0180 0x00145 0x00145 0x00145 0x00145 0x00145" questi per esempio sono quelli per la WB 4


Infine potete settare i parametri di acquisizione:
``` bash
bash daq_run_launch.sh -j -N "0 1 2 3 4 5 6 7 8 9 10 11" -S "start_th" -P "stop_th" -L "lead" -T "tail"
```
**Lead**: campioni da acquisire prima del trigger. Esempio "8 8 8 8 8 8 8 8 8 8 8 8" acquisisce 8 campioni prima del trigger
**Tail**: campioni da acquisire dopo il trigger. Esempio "8 8 8 8 8 8 8 8 8 8 8 8" acquisisce 8 campioni dopo il trigger

**Start_th e Stop_th**: soglia di start e stop. di sotto avete uno scriptino python per ottenere le soglie in esadecimale partendo da quelle in mV (per polarità NEGATIVA, con board 1):
**NB**: Controllare che i valori di calizbrazione siano quelli giusti!!!
``` python
v_to_adc_all = {
  "1n":{0: [14478.60239767,   105.72266231],
     10: [14581.95209726,    94.49772418],
     11: [14585.64212614,    71.52928685],
     2: [14550.19800913,    99.07229959],
     3: [14705.41800247,   120.3148473],
     4: [14543.1833827,   140.8470843],
     5: [14575.71114986,    83.09878558],
     6: [14502.89628843,   125.51115324],
     7: [14756.93105566,   122.49110006],
     8: [14692.74249293,    64.11840958],
     9: [14916.64285453,  -109.3580361],
     1: [14550.19800913,    99.07229959]}}

wvb_active="1n"
ch=1

def convert_v_to_adc(element,ch):
    return (v_to_adc_all[wvb_active][ch][1]+element*v_to_adc_all[wvb_active][ch][0])

stop_th= str(int(0x3fff) - int(convert_v_to_adc(float(stop_th_mv[ch])*0.001,ch)))
print(stop_th)
```

Per l'HV usare il comando ```./SetHv``` e seguire le istruzioni per ogni canale. 


Per far partire una acquisizione, aprire un nuovo terminale e entrare in SSH nella scheda. Eseguire il comando:
```
./DaqReadTcp
```

Su un altro terminale del computer host (quindi NON in SSH) eseguire questo comando per stabilire la connessione:
per salvare su file le forme d'onda:
```
nc 192.168.137.30 5000 > file.bin
```
per vedere la rate su schermo (sostituire a ```delay``` e ```interval``` i valori desiderati):
```
nc 192.168.137.30 5000 | ./RateParser_x86 -a -t "interval" -d "delay" -c 13
```
Infine tornare su un secondo terminale in SSH nella scheda (non lo stesso dove si era lanciato DaqReadTcp) e lanciare il seuente comando per far partire l'acquiszione:

``` bash
bash daq_run_launch.sh -N "0 1 2 3 4 5 6 7 8 9 10 11" -S "start_th" -P "stop_th" -L "lead" -T "tail"
```
Il comando è lo stesso usato per settare i parametri, ma senza il  ```-j``` che inibisce lo start della presa dati. 

Per far smettere la presa dati lanciare:
``` bash
bash daq_run_stop.sh -N "0 1 2 3 4 5 6 7 8 9 10 11"
```


# Ilarizzazione waveboard
*11 Ottobre 2024*

### 0) Verificare posizione jumperino blu "JP1"
[immagine]
- Va messo fra "max 5V" e "P 5 V 0"

![IMG_3094](https://github.com/user-attachments/assets/f9b675bf-208a-4742-ae5b-e637b82b7048)


### 1) Calibrazione in voltaggio (~5min)
*"I canali mi stanno dando lo stesso voltaggio che io gli ho detto di dare via interfaccia?*
- Da un terminale collegato via ethernet alla WB si da
``` bash
ssh root@192.168.137.30
[inserire pwd]
./SetHV -c {n} -p 27.5 -v
```
- Così ho chiesto di settare il dato canale a 27.5V
- Il programma ti chiede di leggere due valori di voltaggio col multimetro e inserirli
![IMG_3092](https://github.com/user-attachments/assets/962a97b9-90ee-428f-8de3-50e9d24d2749)
- Alla fine della procedura il multimetro dovrebbe leggere 27.5V
	- Tuttavia, i vari valori di `m` e `q` per ogni canale vanno copiati nel file `config_ultra.py`, in modo che possano essere caricati all'apertura della GUI
	- Si assume quindi che la calibrazione rimanga costante (a meno di eventi maggiori) in caso di spegni/riaccendi
![IMG_3095](https://github.com/user-attachments/assets/7126365e-24b6-4fa9-afff-10b91157a4a4)


## 2) Calibrazione ADC-Volt (~1h)
- Generare un'onda "pulse" (durata 200ns, frequenza 100Hz) da 100, 200, 300, 400, 800 e 1000 mV, lavorando sempre con attenuatore in 0.1
![IMG_3096](https://github.com/user-attachments/assets/a93ba73a-cf32-4329-83b2-dac477935f90)

- Si manda questa onda nel segnale di un canale per volta, avendo disattivato l'HV, mettendo una soglia di entrata a 5mV
	- Per motivi pratici conviene su un singolo canale fare tutte le 6 onde quadre
- Per ogni configurazione (canale e voltaggio) si acquisiscono N forme d'onda
- Si passano poi queste forme d'onda al programma dedicato (`calwb3_x20.ipynb`) che identifica il plateau per fare la conversione ADC-V
	- Questo programma tira fuori per tutti i canali grafici e coefficienti angolari e intercetta sia Volt-ADC che ADC-Volt
- Questi numeri vanno messi a mano  nel file `config_ultra.py`

## 3) Calibrazione Dark Count (~3h)
*Tutti i canali (collegati ed alimentati) devono contare la stessa cosa in assenza di sorgente*
- Si collegano tutti i rivelatori, ma se ne alimentano metà per volta con soglia 1mV in entrata e 1mV in uscitas
- Senza sorgente, si lancia una acquisizione di ~1h di forme d'onda, creando un file `.bin`
- Questo file va passato dentro l'interfaccia grafica che lo converte in un file  `.txt` per ogni canale con dentro tutto l'elenco delle forme d'onda
- C'è un programma per generare tutti i timestamp di ogni singola forma d'onda
```bash
cd waveboard/bin
./HitViewer_arm -f file.bin -c {n channel} -t >> nome_file_n.txt
```
- Dai timestamp ottieni la rate (perché la WB scrive solo quando riceve qualcosa)
- Dal file `.txt` delle forme d'onda si fa la cumulativa (`nuova_scelta_soglia_buio.ipynb`), e da queste si vede quale soglia (in entrata) scegliere per arrivare ad una dark current di ~0.5CPS
- Ottenuta la soglia, la si imposta sulla GUI, e si fa la verifica con rate monitor
	- La differenza eventuale potrebbe essere dovuta al fatto che si sta impostando solo una delle soglie
- In teoria a questo punto tutti i canali hanno stessa DC, ma non per forza stessi conteggi su sorgente
## 4) Calibrazione su sorgente
*Adattare i Vbias per avere steso conteggio su sorgente*
- Si mettono a coppie i rivelatori sulla sorgente di Sr estesa e si prendono le rate
- Per ciascun canale si trova il Vbias che fa contare quanto il canale di riferimento
