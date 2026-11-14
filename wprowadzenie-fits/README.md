# Wprowadzenie: pliki FITS, WCS, DS9 i stackowanie śladów

Materiały na spotkanie koła. Główna część to notebook `wprowadzenie_fits.ipynb`. Jest zapisany razem z wynikami, więc można go przeglądać bez uruchamiania.

| Sekcja | Temat |
|---|---|
| 1 | Czym jest plik FITS (HDU, nagłówek, dane, zscale) |
| 2 | Nagłówek WCS: piksel ↔ RA/Dec, skala, obrót, projekcja TAN-SIP |
| 3 | DS9: instalacja, komendy, pliki regionów |
| 4 | Galeria przypadków: od pustego nieba do wyraźnego śladu |
| 5 | Usuwanie gwiazd: raw vs preprocessed, mediana „ręcznie” |
| 6 | Shift-and-stack: ślad przed i po stackowaniu, zysk √N |
| 7 | Problemy stackowania: czułość na hipotezę (v, θ), koszt obliczeń |
| 8 | Podsumowanie i zadania |

## Setup

```bash
cd wprowadzenie-fits
uv venv .venv && source .venv/bin/activate      # albo: python -m venv .venv
uv pip install -r requirements.txt               # albo: pip install -r requirements.txt
jupyter lab wprowadzenie_fits.ipynb
```

### DS9: instalacja i otwieranie `.fits` dwuklikiem

```bash
sudo apt install saods9                  # macOS: brew install --cask saoimage-ds9
```

**Linux:** ustaw DS9 jako domyślną aplikację dla plików FITS:

```bash
cat > ~/.local/share/applications/ds9.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=SAOImage DS9
Exec=ds9 %f
MimeType=image/fits;application/fits;
Terminal=false
Categories=Science;
EOF
update-desktop-database ~/.local/share/applications
xdg-mime default ds9.desktop image/fits application/fits
```

Potem dwuklik na `.fits` w menedżerze plików otwiera plik w DS9. Jeśli menedżer nadal pyta, wybierz raz „Otwórz za pomocą → SAOImage DS9 → zawsze”.

**macOS:** w Finderze: prawy klik na `.fits` → „Pobierz informacje” → „Otwórz w programie” → DS9 → „Zmień wszystkie…”.

Przydatne opcje wiersza poleceń (np. do `Exec=`): `ds9 plik.fits -zscale -cmap grey`. Wiele plików w trybie mrugania: `ds9 ramki_fits/slady_syntetyczne/raw/*.fits -zscale -blink`.

Uruchomienie całego notebooka trwa ~20 s i zajmuje ~1.5 GB RAM.

## Dane (`ramki_fits/`)

Ramki FITS **nie są w gicie** (~900 MB). Skopiowano je z `evaluation/dataset/`:

```bash
D=wprowadzenie-fits/ramki_fits
mkdir -p $D/slady_syntetyczne $D/slady_hybrydowe $D/puste_niebo
cp -r evaluation/dataset/obs_009_snr-sweep-3-096-leo-40534/{raw,preprocessed,configs} $D/slady_syntetyczne/
cp -r evaluation/dataset/original/{raw,preprocessed,configs,sim-config.yml} $D/slady_hybrydowe/
rm $D/slady_hybrydowe/configs/config.yml
cp evaluation/dataset/40534_no_objects/raw/c241015_211422_0000_00010_40534__to__c241015_211422_0000_00014_40534_med5.fits \
   $D/puste_niebo/puste_niebo_med5.fits
```

| Folder | Źródło | Ramki | Ślady |
|---|---|---:|---|
| `slady_hybrydowe/` | `original`: prawdziwe ramki z kamery + wstrzyknięte słabe ślady | 11 | 7, SNR 0.07–0.96 |
| `slady_syntetyczne/` | `obs_009_snr-sweep-3-096-leo-40534`: w pełni symulowane | 13 | 6, SNR 2.8–7.3 |
| `puste_niebo/` | `40534_no_objects`: mediana 5 prawdziwych ramek | 1 | brak |

Uwaga: `(x, y)` w `configs/` oznacza w zestawie syntetycznym **środek** śladu, a w hybrydowym **lewy dolny róg** stempla ze śladem (`simulator/simulator/hybrid/streak.py`). Notebook przelicza to w `streak_center()`.

## Pliki generowane

Notebook zapisuje do `wyniki/` (ignorowane przez gita):
- `*_frame0.reg`: regiony DS9 z prawdziwymi pozycjami śladów,
- `single_T2.fits`, `stack_T2.fits`: wycinek pojedynczej ramki i stacku do porównania w DS9.
