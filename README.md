# Лаборатори №6 — Category-Partition ба PICT

**Оюутан:** Н.Мөнхбаяр  
**Код:** B242270058  

---

## Алхам 1: PICT-ийг эх кодоос бүтээх

PICT-ийг Microsoft-ийн GitHub репогоос эх кодоос нь бүтээж, системд суулгасан.

### Хийсэн алхмууд

```bash
# 1. Хамааралтай багцуудыг суулгах
sudo apt update && sudo apt install -y git cmake g++ make

# 2. PICT-ийг лабын репоосоо ГАДНА clone хийх
mkdir -p ~/tools && cd ~/tools
git clone https://github.com/microsoft/pict
cd pict

# 3. CMake-ээр бүтээх
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j4

# 4. Системд суулгах
sudo cp build/cli/pict /usr/local/bin/

