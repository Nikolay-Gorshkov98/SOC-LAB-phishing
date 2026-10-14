# Порядок проведения сценария

Пошаговая инструкция по повторению сценария на стенде. Атакующая машина Kali имеет адрес 192.168.56.102, сервер SOC_LAB имеет адрес 192.168.56.10. Разбор результатов приведён в README.md этого каталога.

Команды на SOC_LAB удобно выполнять по SSH с компьютера-хоста: ssh nikolay@192.168.56.10.

---

## Часть 1. Почтовый сервер (SOC_LAB)

При установке Postfix выбирается тип конфигурации Local only, имя системы soclab.

```bash
sudo apt update
sudo apt install -y postfix mutt
sudo postconf -e "inet_interfaces = all"
sudo postconf -e "mynetworks = 127.0.0.0/8 192.168.56.0/24"
sudo postconf -e "mydestination = soclab, localhost, localhost.localdomain"
sudo systemctl restart postfix
sudo ufw allow from 192.168.56.0/24 to any port 25 proto tcp
```

---

## Часть 2. Контроль каталога загрузок (SOC_LAB)

```bash
mkdir -p ~/Downloads
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
sudo tee -a /var/ossec/etc/ossec.conf > /dev/null <<'EOF'

<ossec_config>
  <syscheck>
    <directories realtime="yes" check_all="yes">/home/nikolay/Downloads</directories>
  </syscheck>
</ossec_config>
EOF
```

---

## Часть 3. Список хэшей и правила (SOC_LAB)

Список создаётся из файла lists/malware-hashes этого репозитория и подключается в основном блоке ruleset конфигурации, сразу после стандартного списка audit-keys:

```bash
echo "44d88612fea8a8f36de82e1278abb02f:EICAR test file" | sudo tee /var/ossec/etc/lists/malware-hashes
sudo chown wazuh:wazuh /var/ossec/etc/lists/malware-hashes
sudo sed -i 's|<list>etc/lists/audit-keys</list>|<list>etc/lists/audit-keys</list>\n    <list>etc/lists/malware-hashes</list>|' /var/ossec/etc/ossec.conf
sudo grep -n -E "<list>|Downloads" /var/ossec/etc/ossec.conf
```

Содержимое rules/local_rules.xml добавляется в конец /var/ossec/etc/rules/local_rules.xml. Затем Wazuh перезапускается:

```bash
sudo systemctl restart wazuh-manager
```

После перезапуска модуль FIM сначала заново сканирует каталоги и только затем включает слежение в реальном времени. Готовность проверяется так:

```bash
sudo grep "Real-time file integrity monitoring started" /var/ossec/logs/ossec.log | tail -n 1
```

---

## Часть 4. Подготовка вложения (Kali)

Строка EICAR записывается без перевода строки, иначе изменится хэш:

```bash
printf '%s' 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > ~/invoice.pdf.exe
ls -l ~/invoice.pdf.exe
md5sum ~/invoice.pdf.exe
```

Ожидается размер 68 байт и MD5 44d88612fea8a8f36de82e1278abb02f.

---

## Часть 5. Отправка письма (Kali)

```bash
swaks --server 192.168.56.10 --to nikolay@soclab --from billing@invoice-portal.test --header "Subject: Invoice for payment (SOC lab test)" --body "SOC lab test message. Attachment is the EICAR test file." --attach-type application/octet-stream --attach-name invoice.pdf.exe --attach @/home/kali/invoice.pdf.exe
```

Ожидается ответ сервера 250 2.0.0 Ok: queued.

Проверка доставки на SOC_LAB:

```bash
sudo journalctl --since today --no-pager | grep -E "invoice-portal|status=sent" | tail -n 5
```

---

## Часть 6. Сохранение вложения (SOC_LAB)

1. Запустить mutt.
2. Выбрать письмо Invoice for payment (SOC lab test) и нажать Enter.
3. Нажать v, чтобы открыть список вложений.
4. Выбрать invoice.pdf.exe и нажать s.
5. Очистить строку комбинацией Ctrl+U, ввести /home/nikolay/Downloads/invoice.pdf.exe и нажать Enter.
6. Выйти клавишей q.

Для проверки правила по двойному расширению создаётся безобидный файл:

```bash
echo "harmless test" > ~/Downloads/report.docx.exe
```

---

## Часть 7. Проверка обнаружения (SOC_LAB)

```bash
sudo grep -A1 -E "^2026.*syscheck" /var/ossec/logs/alerts/alerts.log | grep "Rule:" | tail -n 3
```

Ожидаются алерты 100200 (уровень 13) для invoice.pdf.exe и 100201 (уровень 10) для report.docx.exe.

На дашборде: раздел Discover, индекс wazuh-alerts-*, запрос rule.id:100200 or rule.id:100201.

---

## Возможные проблемы

1. Алертов 100200 и 100201 нет, а есть только 554. Проверьте, что Wazuh перезапущен после добавления правил и что список malware-hashes подключён внутри основного блока ruleset, а не отдельным блоком.
2. Нет даже алерта 554. Подождите, пока в ossec.log появится строка о запуске слежения в реальном времени, и повторите сохранение файла.
3. Хэш не совпадает. Файл EICAR создан с переводом строки: пересоздайте его командой printf из части 4.
4. Письмо не доставляется. Проверьте, что Postfix слушает порт 25 на всех интерфейсах: sudo ss -tlnp | grep ':25 '.
