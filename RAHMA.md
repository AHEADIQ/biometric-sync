# Rahma fork

This is [frappe/biometric-attendance-sync-tool](https://github.com/frappe/biometric-attendance-sync-tool)
(from `a7ce66c`) with one addition: after each device's pull it reports to the Rahma
HIS whether the pull worked, so HR sees on **Attendance & Shifts > Overview** when a
device -- or this tool -- stops working. Everything else is upstream.

## What the addition does

In `erpnext_sync.py`, after every device's turn the tool posts to
`rahma_core.api.biometric_device.report_device_status`:

- `ok`, or the error -- the real one ("timed out", "unauthenticated"...), where
  upstream raised a bare "Device fetch failed.";
- the number of logs pulled and the newest punch on the device;
- `PULL_FREQUENCY` and the host name.

A failure while *posting* punches does not mark the device down: the device
answered. The report never breaks syncing -- 10 s timeout, every error logged to
`logs/error.log` and swallowed. `REPORT_DEVICE_STATUS = False` in `local_config.py`
turns it off.

The HIS marks a device Not Reporting after 2 x `PULL_FREQUENCY` + 10 min without a
report, and when every device goes quiet at once it reports the tool itself stopped.

## Install (Linux server)

Outside the frappe bench, on a machine that reaches the devices (TCP 4370):

```bash
sudo git clone https://github.com/Hasan-Mehdi22/AHEADIQ-biometric-sync.git /opt/biometric-sync
sudo chown -R frappe: /opt/biometric-sync
cd /opt/biometric-sync
python3 -m venv venv && venv/bin/pip install -r requirements.txt
cp local_config.py.template local_config.py   # then fill it in, see below
sudo cp deploy/biometric-sync.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now biometric-sync
```

`local_config.py`:

- `ERPNEXT_URL` -- the HIS, e.g. `http://10.100.60.3`
- `ERPNEXT_API_KEY` / `ERPNEXT_API_SECRET` -- of a dedicated user (e.g.
  `biometric-sync@...`) with create on **Employee Checkin** and write on
  **Shift Type**. Reporting the status needs nothing more.
- `devices` -- one entry per device. `device_id` is the name the device gets in the
  HIS (**Biometric Device**); keep it as it is in the check-ins already synced.
- `PULL_FREQUENCY` -- minutes between pulls.

Check it: `journalctl -u biometric-sync -f` and `logs/error.log`; in the HIS the
device turns Online after its first pull.

## Update

```bash
cd /opt/biometric-sync && git pull && sudo systemctl restart biometric-sync
```

Keep `local_config.py` (it is not in git).

## Taking upstream changes

```bash
git remote add upstream https://github.com/frappe/biometric-attendance-sync-tool.git
git fetch upstream && git merge upstream/master
```

License: GPL-3.0, as upstream.
