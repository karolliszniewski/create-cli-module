# create-cli-module

Pomogę Ci stworzyć moduł w Magento 2, który umożliwi import produktów z pliku CSV poprzez komendę konsolową.
Moduł importu produktów dla Magento 2Click to open code
Aby zaimplementować ten moduł, wykonaj następujące kroki:

Utwórz strukturę katalogów:
```bash
Copyapp/code/Vendor/ImportModule/
├── Console/
│   └── Command/
│       └── ImportProducts.php
├── etc/
│   ├── module.xml
│   └── di.xml
└── registration.php
```

Skopiuj kod z artefaktu do odpowiednich plików zachowując strukturę.
Po utworzeniu wszystkich plików, wykonaj następujące komendy w terminalu:

```bash
bin/magento module:enable Vendor_ImportModule
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:clean
```

Teraz możesz używać komendy do importu:

```bash
bin/magento import:products -f Documents/books3clean.csv
```

Struktura pliku CSV powinna zawierać co najmniej następujące kolumny:

sku
name
price

Możesz dostosować kod w ImportProducts.php do swoich potrzeb, dodając więcej atrybutów produktu w zależności od struktury twojego pliku CSV.
Kilka ważnych uwag:

Zmień namespace Vendor na nazwę swojej firmy/projektu
Dostosuj mapowanie danych w metodzie execute() do struktury twojego pliku CSV
Dodaj obsługę dodatkowych atrybutów produktu według potrzeb
Możesz dodać więcej opcji do komendy (np. tryb testowy, kategorię produktów itp.)

Czy potrzebujesz dodatkowych wyjaśnień odnośnie którejś części kodu lub procesu instalacji?



```php
<?php
// app/code/Vendor/ImportModule/registration.php
\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'Vendor_ImportModule',
    __DIR__
);

// app/code/Vendor/ImportModule/etc/module.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_ImportModule" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Catalog"/>
        </sequence>
    </module>
</config>

// app/code/Vendor/ImportModule/etc/di.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Framework\Console\CommandList">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="import_products" xsi:type="object">Vendor\ImportModule\Console\Command\ImportProducts</item>
            </argument>
        </arguments>
    </type>
</config>

// app/code/Vendor/ImportModule/Console/Command/ImportProducts.php
<?php
namespace Vendor\ImportModule\Console\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Input\InputOption;
use Magento\Framework\App\State;
use Magento\Catalog\Model\ProductFactory;
use Magento\Store\Model\StoreManagerInterface;

class ImportProducts extends Command
{
    protected $state;
    protected $productFactory;
    protected $storeManager;

    public function __construct(
        State $state,
        ProductFactory $productFactory,
        StoreManagerInterface $storeManager
    ) {
        $this->state = $state;
        $this->productFactory = $productFactory;
        $this->storeManager = $storeManager;
        parent::__construct();
    }

    protected function configure()
    {
        $this->setName('import:products')
            ->setDescription('Import products from CSV file')
            ->addOption(
                'file',
                'f',
                InputOption::VALUE_REQUIRED,
                'CSV file path'
            );
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        try {
            $this->state->setAreaCode(\Magento\Framework\App\Area::AREA_ADMINHTML);
            
            $filePath = $input->getOption('file');
            if (!file_exists($filePath)) {
                $output->writeln("<error>File not found: {$filePath}</error>");
                return 1;
            }

            $handle = fopen($filePath, 'r');
            if (!$handle) {
                $output->writeln("<error>Cannot open file</error>");
                return 1;
            }

            // Zakładamy, że pierwsza linia to nagłówki
            $headers = fgetcsv($handle);
            $count = 0;

            while (($data = fgetcsv($handle)) !== false) {
                $productData = array_combine($headers, $data);
                
                try {
                    $product = $this->productFactory->create();
                    
                    // Przykładowe mapowanie danych - dostosuj według swojej struktury CSV
                    $product->setSku($productData['sku'])
                        ->setName($productData['name'])
                        ->setTypeId('simple')
                        ->setAttributeSetId(4) // Default attribute set dla produktów
                        ->setStatus(1) // 1 = enabled
                        ->setVisibility(4) // 4 = visible in catalog and search
                        ->setPrice($productData['price'])
                        ->setStoreId($this->storeManager->getStore()->getId());
                    
                    $product->save();
                    $count++;
                    
                    $output->writeln("<info>Imported product: {$productData['sku']}</info>");
                } catch (\Exception $e) {
                    $output->writeln("<error>Error importing {$productData['sku']}: {$e->getMessage()}</error>");
                }
            }

            fclose($handle);
            $output->writeln("<info>Import completed. Imported {$count} products.</info>");
            
            return 0;
        } catch (\Exception $e) {
            $output->writeln("<error>{$e->getMessage()}</error>");
            return 1;
        }
    }
}
```


