```
sudo apt update
sudo apt upgrade

sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https lsb-release curl -y

curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main | sudo tee /etc/apt/sources.list.d/postgresql.list

sudo apt update
sudo apt install postgresql-client-16 postgresql-16
systemctl status postgresql
sudo systemctl enable postgresql --now
```
```
sudo -i -u postgres
psql
exit
```
