# https://stackoverflow.com/questions/17845637/how-to-change-vagrant-default-machine-name
purpose = "vagrant-asmap"
memsize_gb = ENV['MEMSIZE_GB'] || '12'
force_analysis = ENV['FORCE_ANALYSIS'] || 'false'
force_download = ENV['FORCE_DOWNLOAD'] || 'false'

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.define purpose  
  config.vm.hostname = purpose
  config.vm.provider :virtualbox do |vb|
     vb.name = purpose
     vb.customize ["modifyvm", :id, "--memory", "#{memsize_gb.to_i * 1024}", "--audio", "none"]
     vb.default_nic_type = "Am79c973"
     if Vagrant.has_plugin?("vbguest")
        vb.vbguest.auto_update = true
     end
  end

  config.vm.provision "shell", inline: <<-SHELL

    ### install dependencies
    set -euo pipefail
    export DEBIAN_FRONTEND=noninteractive
    
    [ -f .uptodate ] || {
      apt-get update && touch .uptodate;
    }
   
    apt-get install -y \
      autoconf \
      build-essential \
      cargo \
      clang \
      libboost-filesystem-dev \
      libssl-dev \
      libtool \
      pkg-config \
      tree

    ### checkout rrybarczyk/asmap-rs
    readonly asmap_dir="/vagrant/asmap"
    [ -d "${asmap_dir}" ] || {
      echo "the directory where the vagrant command is called does not contain an asmap sub-directory" >&2
      exit 1
    }
    readonly asmap_rs_dir="${asmap_dir}/asmap-rs"
    [ -f "${asmap_rs_dir}/Cargo.toml" ] || git clone --depth 1 https://github.com/rrybarczyk/asmap-rs.git "${asmap_rs_dir}"

    #### asmap-rs: download files
    pushd "${asmap_rs_dir}"
    dump_directory="${asmap_dir}/dumps"
    mkdir -pv "${dump_directory}"
    for n in $(seq 0 16) $(seq 18 24); do
      echo "checking whether dump file $n download is needed..." >&2
      dump_file_get="cargo run --release download -n $n --out '${dump_directory}'"
      dump_file="${dump_directory}/$(printf "rrc%02d-latest-bview.gz" $n)"            
      if #{force_download} || ! [ -f "${dump_file}" ]; then
        echo "downloading ${dump_file} (forced: #{force_download})..." >&2
        eval "$dump_file_get";
      fi;
      echo "checking ${dump_file} integrity..." >&2
      if [ "${dump_file}" -nt "${dump_file}.checked" ]; then
        gzip --test "${dump_file}" || {
          echo "${dump_file} invalid, downloading..." >&2
          eval "$dump_file_get"
          gzip --test "${dump_file}" || {
            echo "${dump_file} invalid even after one retry, moving on..." >&2
            continue
          }
        }
        touch "${dump_file}.checked"
      fi
    done
    popd # "${asmap_rs_dir}"

    readonly bottleneck_directory="${asmap_dir}/bottlenecks"
    readonly bottleneck_file="$(ls -1c "${bottleneck_directory}" | head -n1)"

    #### asmap-rs: analyze files
    if #{force_analysis} || [ "${bottleneck_file}" == "" ]; then
      pushd "${asmap_rs_dir}"
      echo "analyzing (forced: #{force_analysis}, using #{memsize_gb}GB of RAM)..."
      mkdir -pv "${bottleneck_directory}"
      rm -vf stderr.txt exitcode.txt
      cargo run --release find-bottleneck \
          -d "${dump_directory}" \
          -o "${bottleneck_directory}" \
            > /dev/null 2> stderr.txt  || {
        
        readonly exit_code=$?
        echo "${exit_code}" > exitcode.txt
        tail -v stderr.txt exitcode.txt >&2
        exit ${exit_code}
      }
      popd # "${asmap_rs_dir}"
    fi
    
    readonly bitcoin_dir="${asmap_dir}/bitcoin"

    ### checkout sipa/bitcoin (202004_asmap_tool)
    [ -d "${bitcoin_dir}" ] || {
      echo "checking out sipa/bitcoin..."
      git clone \
        --single-branch \
        --branch 202004_asmap_tool \
          https://github.com/sipa/bitcoin.git \
          "${bitcoin_dir}"
    }

    #### autogen
    [ "${bitcoin_dir}/autogen.sh" -ot "${bitcoin_dir}/configure" ] || {
      echo "creating configure script..."
      pushd "${bitcoin_dir}";
      ./autogen.sh;
      popd; # "${bitcoin_dir}";
    }

    #### configure
    [ "${bitcoin_dir}/configure" -ot "${bitcoin_dir}/Makefile" ] || {
      echo "running configure script..."
      pushd "${bitcoin_dir}";
        ./configure \
          --disable-tests \
          --disable-bench \
          --disable-dependency-tracking \
          --without-daemon \
          --without-libs \
          --without-utils \
          --without-gui \
          --without-miniupnpc \
          --disable-man \
          --without-natpmp \
          --disable-wallet \
          --with-whatever \
          --enable-static \
          --enable-util-asmap;
      popd; # ${bitcoin_dir};
    }

    #### make
    [ -f "${bitcoin_dir}/src/bitcoin-asmap" ] || {
      echo "compiling bitcoin-asmap..."
      pushd "${bitcoin_dir}";
      make;
      popd; # "${bitcoin_dir}";
    }

    readonly asmap_file="${asmap_dir}/ip_asn.map"
    readonly asmap_bottleneck_file="${bottleneck_directory}/${bottleneck_file:-$(ls -1c "${bottleneck_directory}" | head -n1)}"

    #### encode ip_asn.map
    if [ "${asmap_bottleneck_file}" -nt "${asmap_file}" ]; then
      echo "encoding ${asmap_file} from ${asmap_bottleneck_file}..." >&2
      "${bitcoin_dir}/src/bitcoin-asmap" encode "${asmap_file}" < "${asmap_bottleneck_file}"
    fi
    
    ### visualize results
    tree "${asmap_dir}" --prune -hDFP "rrc*.gz|bottleneck*.txt|*asmap|ip_asn.map"
    
    echo "exited normaly"
  SHELL
end

