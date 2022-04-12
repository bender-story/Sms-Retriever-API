# Sms-Retriever-API

```console
#Run the command
#
#chmod u+x sms_retriever_hash_der.sh
#
#If certificate format is .der use
#./sms_retriever_hash_der.sh --alias "your_alias" --package "your_package_name" --cert ./your_der_file_from_play_console.der  --storepass "your_alias_password"
#If certificate format is .keystore use
#./sms_retriever_hash.sh --package "your_package_name" --cert ./app/your_keystore.keystore

VERSION=0.1.0
SUBJECT=sms-retriever-hash-generator
USAGE="Usage: sms_retriever_hash.sh --alias keystore_alias --package package_name --cert cert_file --storepass store_password "

#--- Options processing -------------------------------------------
if [ $# == 0 ]; then
  echo $USAGE
  exit 1
fi

if [[ "$1" != "--package" ]]; then
  echo "Error: expected --package as second parameter"
  exit 1
fi
pkg="$2"
shift 2

if [[ "$1" != "--cert" ]]; then
  echo "Error: expected --cert as third parameter"
  exit 1
fi
cert_file="$2"
shift 2

echo
echo "package name: $pkg"
echo "cert file: $cert_file"
echo

#replace . with _ of in package
package_name=${pkg//./_}

echo "package_name $package_name"

#certificate jks path
cert_path="${package_name}_cert.jks"

#Check if the file is der
if [ "${cert_file: -4}" == ".der" ]; then
  #check if the store password is sent
  if [[ "$1" != "--storepass" ]]; then
    echo "Error: expected --storepass as fourth parameter"
    exit 1
  fi
  store_pass="$2"
  shift 2
  echo "store password : $store_pass"

  #check if the store alias is sent
  if [[ "$1" != "--alias" ]]; then
    echo "Error: expected --alias as first parameter"
    exit 1
  fi
  alias="$2"
  shift 2
  echo "alias name: $alias"

  #Convert der file to jks file using store password and alias
  (keytool -importcert -alias $alias -file $cert_file -keystore $cert_path -storepass $store_pass)
  echo "certificate path: $cert_path"

  #Get the certificate
  cert=$(keytool -exportcert -alias "$alias" -keystore $cert_path | xxd -p | tr -d "[:space:]")

  echo "certificate in hex: $cert"
  #concatenate input
  input="$pkg $cert"

  #256 bits
  output=$(printf "$input" | shasum -a 256 | tr -d "[:space:]-")
  echo
  echo "SHA-256 output in hex: $output"

else
  #if cert file type is keystore use this
  cert=$(keytool -list -rfc -keystore $cert_file | sed -e '1,/BEGIN/d' | sed -e '/END/,$d' | tr -d ' \n' | base64 --decode | xxd -p | tr -d ' \n')

  echo
  echo "certificate in hex: $cert"

  #concatenate input
  input="$pkg $cert"

  #256 bits = 32 bytes = 64 hex chars
  output=$(printf "$input" | shasum -a 256 | cut -c1-64)
  echo
  echo "SHA-256 output in hex: $output"

  #take the beginning 72 bits (= 9 bytes = 18 hex chars)
  output=$(printf $output | cut -c1-18)
fi

##encode sha256sum output by base64 (11 chars)
base64output=$(printf $output | xxd -r -p | base64 | cut -c1-11)
echo
echo "First 8 bytes encoded by base64: $base64output"
echo
echo "SMS Retriever hash code:  $base64output"
echo
```
