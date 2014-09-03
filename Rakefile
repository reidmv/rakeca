require 'erb'

RAKE_DIR    = Rake.application.original_dir
ROOT_CA_DIR = File.join(RAKE_DIR, 'rootca_ssldir')

def ca
  ca_dir = File.join(RAKE_DIR, 'ca', 'root')
  case block_given?
  when true
    Dir.chdir(ROOT_CA_DIR) do
      yield
    end
  when false
    ROOT_CA_DIR
  end
end

def dirs
  [ ca,
    File.join(RAKE_DIR, 'certificate_requests'),
    File.join(RAKE_DIR, 'certs'),
    File.join(RAKE_DIR, 'private_keys'),
    File.join(RAKE_DIR, 'public_keys')
  ]
end

def log(msg)
  puts ">>> %s" % msg
end

def ask(prompt, env, default="")
  return ENV[env] if ENV.include?(env)

  if default
    print "#{prompt} (#{default}): "
  else
    print "#{prompt}: "
  end

  resp = STDIN.gets.chomp
  resp.empty? ? default : resp
end

def has_ca?
  File.exist?("#{ca}/serial") && File.exist?("#{ca}/private")
end

def opensslcnf
  File.join(RAKE_DIR, 'openssl.cnf')
end

desc "Create a new CA from scratch"
task :init => [:genca, :gencrl, :genpub]

desc "Create a new CSR and private key"
task :gencsr do
  abort "Please specify a cert name to generate using CN=mycert" unless ENV["CN"]

  sh "openssl genrsa -out private_keys/#{ENV['CN']}.pem 2048"
  sh "openssl req -out certificate_requests/#{ENV['CN']}.pem -new -key private_keys/#{ENV['CN']}.pem -config #{opensslcnf}"
end

desc "Revoke a certificate"
task :revoke do
  abort "Please create a CA using 'rake init'" unless has_ca?
  abort "Please specify a cert to revoke using CN" unless ENV["CN"]
  abort "Cannot find the certificate '#{ENV['CN']}' to revoke" unless File.exist?(ENV['CN'])

  log "Revoking certificate #{ENV['CN']}"

  ca do
    sh "openssl ca -config #{opensslcnf} -revoke '#{ENV['CN']}'"
  end

  Rake::Task["gencrl"].invoke
end

desc "Sign *.csr files"
task :sign do
  abort "Please create a CA using 'rake init'" unless has_ca?

  ENV['CN'] = 'certificate_requests'
  Dir.glob("#{RAKE_DIR}/certificate_requests/*.pem").each do |csr|
    certname = File.join(RAKE_DIR, 'certs', "#{File.basename(csr)}")
    log "Signing #{csr} creating #{certname}"
    ca { sh "openssl ca -batch -config #{opensslcnf} -in #{csr} -out #{certname}" }
    FileUtils.rm csr if File.exist?(certname)
  end
end

desc "Generate the CA"
task :genca do
  abort "CA has already been created in #{ca}/" if has_ca?

  dirs.each do |dir|
    log "Creating directory #{dir}"
    FileUtils.mkdir_p(dir)
  end

  ca do
    ['requests', 'signed', 'private'].each do |dir|
      log "Creating directory #{dir}"
      FileUtils.mkdir dir
    end

    FileUtils.chmod(0700, "private")
    FileUtils.touch("inventory.txt")

    File.open("serial", "w") {|f| f.puts "01"}

    ENV['CN'] = "Puppet Labs Root Certificate Authority #{Time.now.strftime("%s")}"
    sh "openssl req -nodes -config #{opensslcnf} -days 1825 -x509 -newkey rsa:2048 -out ca_crt.pem -outform PEM"
  end
end

desc "Generate the certificate revocation list"
task :gencrl do
  abort "Please create a CA using 'rake init'" unless has_ca?
  ca do
    sh "openssl ca -config #{opensslcnf} -gencrl -out ca_crl.pem"
  end
end

desc "Generate the CA public key"
task :genpub do
  ca do
    sh "openssl rsa -in ca_key.pem -pubout > ca_pub.pem"
  end
end

desc "Completely irreversibly destroy the CA"
task :destroy_ca do
  confirm = ask("Type 'yes' to destroy the CA", "", nil)
  abort "aborting" unless confirm == 'yes'

  [dirs, File.join(RAKE_DIR, 'ca')].flatten.each do |dir|
    log "Removing directory #{dir}"
    FileUtils.rm_rf(dir)
  end
end
