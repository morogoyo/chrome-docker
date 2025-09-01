# Load environment variables from .env file (manually)
env_file = File.join(File.dirname(__FILE__), '.env')
if File.exist?(env_file)
  File.readlines(env_file).each do |line|
    next if line.strip.empty? || line.strip.start_with?('#')
    key, value = line.strip.split('=', 2)
    ENV[key] = value if key && value
  end
end

# Vagrant configuration starts here
Vagrant.configure("2") do |config|

  config.vm.box = "bento/ubuntu-22.04"

  # Optional ENV-based paths
  host_folder = ENV['HOST_SHARED_FOLDER']
  host_folder_ansible = ENV['HOST_SHARED_FOLDER_ANSIBLE']

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 6
    vb.gui = true
  end

  config.vm.boot_timeout = 1800

  config.vm.provision "shell", inline: <<-SHELL
    echo "******************** / System Prep / ********************"
    sudo apt-get update -y
    sudo apt-get install -y python3 apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release net-tools openssh-server git

    echo "******************** / Docker Installation / ********************"
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
      https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update -y
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    sudo systemctl enable docker
    sudo systemctl start docker

    echo "******************** / Create ansible user / ********************"
    sudo groupadd -f docker
    sudo useradd -m -s /bin/bash ansible || true
    echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
    sudo usermod -aG docker ansible

    echo "******************** / Prepare SSH folder / ********************"
    sudo mkdir -p /home/ansible/.ssh
    sudo chown -R ansible:ansible /home/ansible/.ssh
    sudo chmod 700 /home/ansible/.ssh

    echo "******************** / Generate SSH key for internal communication / ********************"
    sudo -u ansible bash -c '
      KEY_PATH="/home/ansible/.ssh/id_rsa_ansible"
      if [ ! -f "$KEY_PATH" ]; then
        ssh-keygen -t rsa -b 4096 -C "ansible@vagrant" -f "$KEY_PATH" -N ""
        chmod 600 "$KEY_PATH"
        chmod 644 "$KEY_PATH.pub"
        echo "✅ SSH key created."
        echo "🔑 Public key (use this for remote nodes or GitHub if needed):"
        cat "$KEY_PATH.pub"
      fi
    '

    echo "******************** / Optional Docker Login from .env / ********************"
    if [ -f /vagrant/.env ]; then
      export $(grep DOCKER_USERNAME /vagrant/.env | xargs)
      export $(grep DOCKER_PASSWORD /vagrant/.env | xargs)
      echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
    else
      echo "/vagrant/.env not found. Skipping Docker login."
    fi

    echo "******************** / Pull and Run Ansible Docker Container / ********************"
    docker pull morogoyo/ansible:dev
    docker run -d --name ansible \
      -u ansible \
      -v /home/ansible/.ssh:/home/ansible/.ssh:ro \
      morogoyo/ansible:dev

  SHELL

  # Optional synced folders for Ansible project or shared code
  config.vm.synced_folder "e:/development/devops", "/home/ansible/devops"

end
