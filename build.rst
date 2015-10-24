# Compile Electrum (master) on a fresh MacOS X Yosemite (10.10.5) install

## Install Brew (http://brew.sh)
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## Update brew
```
brew update
```

## Install python (and thus pip) and pyqt-4
```
brew install python pyqt
```

## Upgrade pip (optional)
```
pip install --upgrade pip
```

## Upgrade all pip packages (optional)
```
pip install pip-review
pip-review --local --interactive
# Some may need sudo... use:
# sudo pip install <package_name> --upgrade
```

## Clone Electrum from the official git repository
```
# You can also get the latest stable released version instead: https://github.com/spesmilo/electrum/releases
git clone https://github.com/spesmilo/electrum.git
cd electrum
```

## Install the Electrum module(s)
```
python setup.py sdist
pip install --pre dist/Electrum-*tar.gz
```

## You may want to add the following modules as well

'#' are optionals
```
brew install gmp zbar # Required for gmpy (pip) and zbar (pip)

pip install certifi cffi configparser crypto cryptography dnspython ecdsa gi gmpy html http jsonrpclib mercurial numpy ordereddict packaging pip ply pyOpenSSL pyasn1 pyasn1-modules pycparser pycrypto setuptools setuptools-svn simplejson wincertstore zbar # Some might be optional

pip install cython trezor # Trezor support, only these two are needed (no need to have limbs or pyusb)

#brew install homebrew/python/matplotlib # Required for Plot History (do not install via pip or Electrum will not compile)
brew install homebrew/python/pillow # Needed for PIL imports as PIL is now depreciated (do not install via pip or Electrum will not compile)
# pip install amodem # Audio Modem plugin (does not work for OS X)
```

## Link all apps
```
brew linkapps
```

## Generate icons
```
pyrcc4 icons.qrc -o gui/qt/icons_rc.py
```

## Compile the Electrum application
```
ARCHFLAGS="-arch i386 -arch x86_64" sudo python setup-release.py py2app --includes sip
```

## Build the dmg container
```
sudo hdiutil create -fs HFS+ -volname "Electrum" -srcfolder dist/Electrum.app dist/electrum-VERSION-macosx.dmg
```

---------

# Things that need to be fixed atm

## Incorrect ca_path

IN paymentrequest.py, line 44:
```
ca_path = requests.certs.where()
```

FIX:
Which can be patched with (temporary and dirty solution):
```
#!/bin/sh

echo "[...] Patching: cacert.pem"

cp -f build/bdist.macosx-*/python2.7-standalone/app/collect/certifi/cacert.pem dist/Electrum.app/Contents/Resources/lib/python2.7/ &&\
chmod 755 dist/Electrum.app/Contents/Resources/lib/python2.7/cacert.pem &&\
sed -i.bak "s/requests.certs.where()/os.path.join(os.path.dirname(__file__), '..\/cacert.pem')/g" dist/Electrum.app/Contents/Resources/lib/python2.7/lib/paymentrequest.py &&\
rm -f dist/Electrum.app/Contents/Resources/lib/python2.7/lib/paymentrequest.py.bak &&\
echo "[OK] Patch applied successfully"
```
A proper solution would be to investigate how ca_path is obtained.

It's the way py2app packages the libs but I don't know how to change that to have a directory instead of a zip archive. Similar issue also reported here: http://stackoverflow.com/questions/28073033/running-pytest-on-module-inside-site-packages-zip

## Invalid DEFAULT_CA_BUNDLE_PATH

IN electrum.py, line 403:
```
assert os.path.exists(requests.utils.DEFAULT_CA_BUNDLE_PATH)
```

FIX:
Can be fixed by commenting the line (which is a temporary and dirty solution). A correct solution is to dig where the DEFAULT_CA_BUNDLE_PATH is set and correct it.

```
#!/bin/sh

echo "[...] Patching: DEFAULT_CA_BUNDLE_PATH"

sed -i.bak "s/assert *os.path.exists(requests.utils.DEFAULT_CA_BUNDLE_PATH)/#assert os.path.exists(requests.utils.DEFAULT_CA_BUNDLE_PATH)/g" dist/Electrum.app/Contents/Resources/electrum.py &&\
rm -f dist/Electrum.app/Contents/Resources/electrum.py.bak &&\
echo "[OK] Patch applied successfully"
```

## Crash when clicking on the qrcode icon in the "Pay to" field of the "Send" tab

FIX:
Remove this feature to scan QR-Codes because it does not work on OSX.

## Plot History plugin doesn't work

The Plot History plugin does not work properly.

In "Export History", when clicking on "Preview plot":
```
Jul 21 12:02:00 dev.local electrum[61643] <Notice>: Traceback (most recent call last):
Jul 21 12:02:00 dev.local electrum[61643] <Notice>:   File "./Electrum.app/Contents/Resources/lib/python2.7/plugins/plot.py", line 42, in <lambda>
Jul 21 12:02:00 dev.local electrum[61643] <Notice>:     b.clicked.connect(lambda: self.do_plot(self.wallet, history))
Jul 21 12:02:00 dev.local electrum[61643] <Notice>:   File "./Electrum.app/Contents/Resources/lib/python2.7/plugins/plot.py", line 58, in do_plot
Jul 21 12:02:00 dev.local electrum[61643] <Notice>:     tx_hash, confirmations, value, timestamp = item
Jul 21 12:02:00 dev.local electrum[61643] <Notice>: ValueError: too many values to unpack
```

FIX:
???

## Preference panel does not work

The application crashes with "terminated by signal SIGSEGV (Address boundary error)".

Crash is due to this import (OS X does not seem to support video at all for qrscanner):
```
from electrum import qrscanner
```

FIX:

```
#!/bin/sh

echo "[...] Patching: Preference pane (qrscanner)"

sed -i.bak -n -e '/^ *def *read_tx_from_qrcode(self):/{' -e 'p' -e ':a' -e 'N' -e '/self.show_transaction(tx)/!ba' -e 's/.*\n/        return #/' -e '}' -e 'p' dist/Electrum.app/Contents/Resources/lib/python2.7/gui/qt/main_window.py &&\
sed -i.bak -n -e '/^ *from *electrum *import *qrscanner/{' -e ':a' -e 'N' -e '/gui_widgets.append((qr_label, *qr_combo))/!ba' -e 's/.*\n/#/' -e '}' -e 'p' dist/Electrum.app/Contents/Resources/lib/python2.7/gui/qt/main_window.py &&\
rm -f dist/Electrum.app/Contents/Resources/lib/python2.7/gui/qt/main_window.py.bak &&\
echo "[OK] Patch applied successfully"
```

## [Errno 20] Not a directory

Plugins Labels and Exchange Rate seem to be impacted with this issue.

This issue is caused by requests.request which returns:
```
[Errno 20] Not a directory
```

Maybe because requests is not properly installed, I don't know :(.

FIX:
???

## Audio MODEM

Does not work (activation fails) because MacOS Kernel 'Darwin' seems not to be supported

```
Jul 21 12:25:18 dev.local electrum[80139] <Notice>: Audio MODEM is available.
Jul 21 12:25:18 dev.local electrum[80139] <Notice>: Traceback (most recent call last):
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:   File "./Electrum.app/Contents/Resources/lib/python2.7/gui/qt/main_window.py", line 2799, in <lambda>
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:     return lambda: do_toggle(cb, name, w)
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:   File "./Electrum.app/Contents/Resources/lib/python2.7/gui/qt/main_window.py", line 2789, in do_toggle
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:     plugins[name] = p = module.Plugin(self.config, name)
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:   File "./Electrum.app/Contents/Resources/lib/python2.7/plugins/audio_modem.py", line 36, in __init__
Jul 21 12:25:18 dev.local electrum[80139] <Notice>:     }[platform.system()]
Jul 21 12:25:18 dev.local electrum[80139] <Notice>: KeyError: 'Darwin'
```

---------

# Other patches (optionals):

## Privacy headers_url

Remove headers_url to prevent the client to download the header file from a centralized/untrusted server (no offense):

```
#!/bin/sh

echo "[...] Patching: headers_url"

sed -i.bak "s/self.headers_url *= *'.*'/self.headers_url = ''/g" dist/Electrum.app/Contents/Resources/lib/python2.7/lib/blockchain.py &&\
rm -f dist/Electrum.app/Contents/Resources/lib/python2.7/lib/blockchain.py.bak &&\
echo "[OK] Patch applied successfully"
```

## Privacy DEFAULT_SERVERS

Remove all (untrusted) DEFAULT_SERVERS:
```
#!/bin/sh

echo "[...] Patching: DEFAULT_SERVERS"

sed -i.bak -n -e '/^ *DEFAULT_SERVERS *= *{/{' -e 'p' -e ':a' -e 'N' -e '/}$/!ba' -e 's/.*\n//' -e '}' -e 'p' dist/Electrum.app/Contents/Resources/lib/python2.7/lib/network.py &&\
rm -f dist/Electrum.app/Contents/Resources/lib/python2.7/lib/network.py.bak &&\
echo "[OK] Patch applied successfully"
```

Note: You'll either need to add some manually to this list or add yours to your Electrum ~/.electrum/config file ("server": "your server.com:50002:s",) otherwise the application will not launch if this is the first time you use it. Also make sure to remove the ~/.electrum/recent_servers file (to avoid your client to connect to previous servers).
