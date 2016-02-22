wget http://ftp.tw.debian.org/debian/dists/testing/main/installer-amd64/current/images/hd-media/boot.img.gz
sudo sh -c "zcat boot.img.gz > /dev/sdb"
wget http://ftp.tw.debian.org/debian-cdimage/daily-builds/testing/current/amd64/iso-cd/debian-testing-amd64-businesscard.iso
