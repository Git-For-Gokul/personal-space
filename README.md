curl -L https://registry.npmjs.org/exceljs/-/exceljs-4.4.0.tgz -o exceljs-4.4.0.tgz

rm -rf node_modules/exceljs
mkdir -p node_modules/exceljs
tar -xvzf exceljs-4.4.0.tgz -C node_modules/exceljs --strip-components=1

