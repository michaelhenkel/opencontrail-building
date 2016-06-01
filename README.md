# opencontrail-building


##apt-get -y install libxml2-utils python-lxml autoconf automake libtool patch unzip pkg-config javahelper ant python-setuptools libboost-dev libboost-chrono-dev libboost-date-time-dev libboost-filesystem-dev libboost-program-options-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev libcurl4-openssl-dev google-mock libgoogle-perftools-dev liblog4cplus-dev libtbb-dev libhttp-parser-dev libxml2-dev libicu-dev python lxml autoconf automake bzip2 libtool patch unzip wget scons git python-lxml wget gcc patch make unzip flex bison g++ libssl-dev autoconf automake libtool pkg-config vim python-dev python-setuptools libprotobuf-dev protobuf-compiler libsnmp-python librdkafka-dev libboost-dev libboost-chrono-dev libboost-date-time-dev libboost-filesystem-dev libboost-program-options-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev libcurl4-openssl-dev google-mock libgoogle-perftools-dev liblog4cplus-dev libtbb-dev libhttp-parser-dev libxml2-dev libicu-dev python-lxml autoconf automake bzip2 libtool patch unzip wget scons git python-lxml wget gcc patch make unzip flex bison g++ libssl-dev autoconf automake libtool pkg-config vim python-dev python-setuptools libprotobuf-dev protobuf-compiler libsnmp-python librdkafka-dev debhelper libxml2-utils python-all python-sphinx ruby-ronn module-assistant ant default-jdk javahelper libcommons-codec-java libhttpcore-java liblog4j1.2-java nodejs python-all python-sphinx ruby-ronn libipfix cassandra-cpp-driver cassandra-cpp-driver-dev libzookeeper-mt-dev libuv librdkafka1 librdkafka-dev libipfix-dev module-assistant python-fixtures python-pydot
#dpkg-buildpackage -S -rfakeroot -k7F0C32CB
#dpkg-source --commit
#dpkg-buildpackage -rfakeroot -us -uc
#dput -f ppa:mhenkel-3/opencontrail libuv1_1.9.0-1_source.changes
#SCONSFLAGS="-j ${JOBS} -Q debug=1" dpkg-buildpackage -b -rfakeroot -k${KEY}


KEY=7F0C32CB
JOBS=32
CONTRAIL_VNC_REPO=git@github.com:Juniper/contrail-vnc.git
CONTRAIL_BRANCH=R3.0
VERSION=3.1.0.0~2733

[ ! -d build ] && mkdir build
cd build
if [[ "$CONTRAIL_BRANCH" == "default" ]]; then
     repo init -u $CONTRAIL_VNC_REPO
else
    repo init -u $CONTRAIL_VNC_REPO -b $CONTRAIL_BRANCH
fi
repo sync

cd third_party
python fetch_packages.py
cd ..

chmod +w packages.make
grep $KEY packages.make
if [ $? -eq 1 ]; then
  sed -i "s/KEYID?=/KEYID?=$KEY/g" packages.make
fi
echo -e "VERSION=$VERSION\n$(cat packages.make)" > packages.make
sed -i "s#| sed 's/tools\\\/packages//'##g" packages.make
sed -i “s#cp -r -a contrail-web-controller/webroot#cp -r -a \${SB_TOP}/contrail-web-controller/webroot#g” ./tools/packages/debian/contrail-web-controller/debian/rules
sed -i '/libipfix,/ a \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ libipfix-dev,' tools/packages/debian/contrail/debian/control
sed -i '/override_dh_install:/ a \\tcp -r -a ${SB_TOP}\/contrail-web-core\/build-files.sh ${INSTALL_ROOT}' tools/packages/debian/contrail-web-core/debian/rules
grep '^source-package-.*:' packages.make |grep -v ceilometer| cut -d : -f 1 | while read i; do
    make -f packages.make $i
done

cd build/packages
for i in *.dsc; do pkgname=$(echo $i|cut -d "_" -f 1); echo $pkgname;mv ${pkgname}*.gz ${pkgname}*.dsc ${pkgname}*.changes ${pkgname}; done
for i in *.dsc; do pkgname=$(echo $i|cut -d "_" -f 1); mv ${pkgname}_*.gz ${pkgname}_*.dsc ${pkgname}/; done

for i in `echo */`; do cd $i;  SCONSFLAGS="-j ${JOBS} -Q debug=1" dpkg-buildpackage -b -rfakeroot -k${KEY}; cd ..; done

cp -r *.deb /var/www/contrail/amd64
cd /var/www/contrail/
dpkg-scanpackages amd64 | gzip -9c > amd64/Packages.gz
apt-get update
