# 3-tier-application-deploye-using-3different-ec2-instance

1.Architecture: 
<img width="614" height="288" alt="image" src="https://github.com/user-attachments/assets/285e9453-0a10-47f7-b7ec-876b6a735e74" />

<img width="1547" height="352" alt="image" src="https://github.com/user-attachments/assets/a7104f8b-79f9-44b5-8fd1-0280ac7da365" />



I created presentation layer instance under presentation layer subnet .
In the same way , application layer instance under application -layer-subnet
And database layer instance under database-layer-subnet


Database setup:


sudo apt update && sudo apt upgrade -y

sudo apt install -y git curl wget unzip build-essential

cd ~

git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git        

cd single-server-3tier-webapp

sudo apt install -y postgresql 

sudo systemctl start postgresql

sudo systemctl enable postgresql

sudo -u postgres psql -c "CREATE USER bmi_user WITH PASSWORD 'your_password';"  

sudo -u postgres psql -c "CREATE DATABASE bmidb;"

sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE bmidb TO bmi_user;"  

sudo -u postgres psql -d bmidb -c "GRANT ALL ON SCHEMA public TO bmi_user;"     

sudo -u postgres psql -d bmidb -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO bmi_user;"

PG_HBA=$(sudo -u postgres psql -t -P format=unaligned -c 'SHOW hba_file')

sudo cp "$PG_HBA" "${PG_HBA}.backup"


sudo sed -i "/^# IPv4 local connections:/a host    bmidb    bmi_user    10.0.255.173/32    md5" "$PG_HBA"

sudo nano /etc/postgresql/18/main/postgresql.conf
//edit 
#listen_addresses = 'localhost’ to
 listen_addresses = '*''
      
      18.sudo ss -tuln | grep 5432

      19.sudo systemctl reload postgresql








These commands perform a series of tasks that I carry out, including updating my system, installing necessary packages, cloning a web application repository, setting up PostgreSQL with remote access, configuring PostgreSQL for connections from specific IPs, and testing the database connectivity. Finally, I clear the command history for security or privacy reasons. 



  Backend Setup:



sudo apt update && sudo apt upgrade -y

sudo apt install -y git curl wget unzip build-essential

cd ~

git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git        

cd single-server-3tier-webapp

curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -

sudo apt install -y nodejs

sudo npm install -g pm2

node -v && npm -v && pm2 -v

sudo mkdir -p /opt/bmi-app/backend
sudo chown -R $USER:$USER /opt/bmi-app

cat > /opt/bmi-app/backend/.env << EOF
DATABASE_URL=postgresql://bmi_user:your_password@10.0.0.4:5432/bmidb
DB_USER=bmi_user
DB_PASSWORD=your_password
DB_NAME=bmidb
DB_HOST=10.0.0.4
DB_PORT=5432
PORT=3000
NODE_ENV=production
CORS_ORIGIN=*
EOF
chmod 600 /opt/bmi-app/backend/.env

rsync -a --exclude 'node_modules' --exclude '.env' --exclude 'logs' ~/single-server-3tier-webapp/backend/ /opt/bmi-app/backend/

cd /opt/bmi-app/backend

npm install --production

PGPASSWORD=your_password psql -U bmi_user -d bmidb -h 10.0.0.4 -f migrations/001_create_measurements.sql

PGPASSWORD=your_password psql -U bmi_user -d bmidb -h 10.0.0.4 -f migrations/002_add_measurement_date.sql


pm2 start src/server.js --name bmi-backend --env production

pm2 save

sudo env PATH=$PATH:$(which node) $(which pm2) startup systemd -u $USER --hp $HOME

pm2 save


Presentation layer setup


sudo apt update && sudo apt upgrade -y

sudo apt install -y git curl wget unzip build-essential

cd ~

git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git        

cd single-server-3tier-webapp

sudo apt install -y nginx

sudo systemctl start nginx

sudo systemctl enable nginx

cd ~/single-server-3tier-webapp/frontend

npm install

npm run build

sudo mkdir -p /var/www/bmi-health-tracker

sudo rm -rf /var/www/bmi-health-tracker/*

sudo cp -r dist/* /var/www/bmi-health-tracker/

sudo chown -R www-data:www-data /var/www/bmi-health-tracker

sudo chmod -R 755 /var/www/bmi-health-tracker

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

EC2_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)

echo $EC2_IP






Nginx configuration:


sudo tee /etc/nginx/sites-available/bmi-health-tracker > /dev/null << 'EOF'
server {
    listen 80;
    server_name _;

    root /var/www/bmi-health-tracker;
    index index.html;

    # React SPA — serve index.html for all non-file routes
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to Node.js backend
    location /api/ {
        proxy_pass http://10.0.255.173:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
}
EOF


sudo ln -sf /etc/nginx/sites-available/bmi-health-tracker /etc/nginx/sites-enabled/bmi-health-tracker

sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t && sudo systemctl restart nginx



Connectivity check:

Database and backend

<img width="798" height="76" alt="image" src="https://github.com/user-attachments/assets/d3c200d3-4843-4983-88f1-7cd8342c9f7e" />




Frontend and backend connectivity:
<img width="1591" height="819" alt="image" src="https://github.com/user-attachments/assets/11685fbb-a4ca-45d6-8743-ab47ea0fef62" />


