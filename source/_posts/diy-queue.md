---
title: DIY Queue Service for Async Messaging
date: 2024-07-14 21:53:13
tags: [queue, async, typescript, web scraping]
---

![abstract queue](/images/queue/async_queue.jpeg "ai generated abstract queue image")

## Background Story

Sometime in January, I needed to scrape about 10k assets(images/texts) and data for some nft data project I had ongoing at the time. The data were unstructured, about 6-7 different varieties in total. At the end, I need to insert these 10k data into an excel sheet and a json file.  
A manual approach to this isn't viable, if I needed to scrape these data, I had to build a system in place that is fault tolerable due to the following:  

- network latency (the network speed in my area is not to be relied on)
- power (same as network)
- accuracy (ensure the file names/descriptions matches the specific file )
- efficiency (scrape and organize in a short time)

So with these as my functional requirements, I had to come up with the best approach to solve my problem.

### Webscraping 101

I use typescript with [nest.js](https://nestjs.com/) for most of my everyday project because of the familiarity and easy setup.
So let's do some webscraping before building our queue service.

### dependencies

`puppeteer:` for webscraping, read the [docs](https://pptr.dev/) for more info.  
`excel-js:` to convert data to excel, check [docs here](https://www.npmjs.com/package/exceljs).  
`@tensorflow-models/mobilenet` and `tensorflow/tfjs-node:` build on top Tensorflow-js, for some image classification, see the [docs here](https://js.tensorflow.org/api/latest/)

#### let's do some basic scraping

```ts
@Injectable()
export class ScrapperService {

    url: string = 'the_web_url_you_want_to_scrape_from';

    get_path = `add_your_chrome's_path_here`;
    
    private readonly scrapedData: any[] = [];

    private readonly logger = new Logger(ScrapperService.name);

    constructor(private queueService: QueueService) {
        this.loadModel(); //load image classification model if any
    }

    private async loadModel() {
    // Assuming you have your model loading logic here
    // this.model = await tf.loadGraphModel(
    //   add the path to your model
    // );
  }

  async processID(id: string) { // process single ID, method takes in the ids of the data to be scrapped
    try {
      this.logger.log('starting to scrape');
      //allow puppeteer to scrape
      const browser = await puppeteer.launch({
        args: ['--no-sandbox'],
        headless: true,
        ignoreHTTPSErrors: true,
        executablePath: this.get_path,
      });
      const addInscriptionId: string = `${this.url}${id}`;
      console.log(addInscriptionId);
      const page = await browser.newPage();
      await page.setUserAgent(
        'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
      );

      page.setDefaultNavigationTimeout(2 * 60 * 1000);

      await Promise.all([
        page.waitForNavigation(),
        page.goto(addInscriptionId),
      ]);
      const titles = await page.$eval(
        '.dogescription-title',
        (elements: any) => {
          return elements.textContent.trim();
        },
      );
      const spanText = await page.evaluate(() => {
        const span: any = document.querySelector('span[data-v-5edf691f]');
        return span ? span.textContent.trim() : null;
      });
      const imageUrl = await page.$eval(
        'img.dogescription-picture',
        (img: any) => img.getAttribute('src'),
      );

      // if images and titles are found
      // you could handle logic to check if they match or not
      if (titles && imageUrl) {
        const imageBuffer = await page.goto(imageUrl);
        const scriptDirectory = __dirname;
        const imageFolderPath = path.join(
          scriptDirectory,
          '..',
          '..',
          '..',
          'scrapped-images',
        );
        await fs.promises.mkdir(imageFolderPath, { recursive: true }); // create the folder if not available
        const filename = path.join(
          imageFolderPath,
          `${titles.replace(/\s+/g, '_')}.png`,
        );
        if (imageBuffer) {
          fs.writeFileSync(filename, await imageBuffer.buffer(), 'base64');
          console.log(`Image for ${id} saved as ${filename}`);
        } else {
          console.error(`Failed to download image for ${id}.`);
        }
      } else {
        console.error(`Title or image URL not found for ${id}.`);
      }
      await browser.close();

      const save2Json = {
        inscriptionId: id,
        name: titles,
        number: spanText,
        imageUrl: imageUrl,
        collectionSymbol: 'doginalsdragon',
        attributes: {
          rank: 'string',
          wings: 'string',
          neck: 'string',
          mouth: 'string',
          shirt: 'string',
          eye: 'string',
        },
      };
      this.scrapedData.push(save2Json);
      return save2Json;
    } catch (err) {
      return runMeErrorHelper(err);
    }
  }

  // handle batch processing of ids[] in array format 500 per batch
  async processArrayIds(ids: string[]): Promise<void> {
    const totalId = ids.length;
    let processedCount = 0;

    try {
      for (const id of ids) {
        this.logger.log(
          `added new id to queue for processing ${id} of ${totalId}`,
        );
        this.queueService.enqueue(id);
      }

      while (!this.queueService.isEmpty()) {
        const id = this.queueService.dequeue();
        if (id) {
          const startTime = performance.now(); // Record start time

          try {
            await this.processID(id);
            processedCount++;

            const endTime = performance.now(); // Record end time
            const elapsedTime = endTime - startTime; // Calculate elapsed time in milliseconds

            this.logger.log(
              `processed ${processedCount} of ${totalId}, ${this.queueService.getSize()} left. Time taken: ${elapsedTime.toFixed(2)} ms for ID ${id}`,
            );
          } catch (err) {
            console.error(`Error processing ID ${id}:`, err);
            // if processing of ids fails, saved already processed data
            await this.saveScrapedDataToJson();
          }
        } else {
          this.logger.log(
            'the ids have finished processing, you can add more for processing',
          );
        }
      }
      await this.saveScrapedDataToJson();
    } catch (err) {
      console.log('we found an error here: ', err);
      return runMeErrorHelper(err);
    }
  }

  async saveScrapedDataToJson(): Promise<void> {
    const jsonFilePath = path.join(
      __dirname,
      '..',
      '..',
      '..',
      'data.json',
    );
    fs.writeFileSync(
      jsonFilePath,
      JSON.stringify(this.scrapedData, null, 2),
      'utf-8',
    );
  }

  // create excel from json data
  async moveJsonToExcel() {
    console.log('hello');
    try {
      const jsonFilePath = path.join(
        __dirname,
        '..',
        '..',
        '..',
        'data.json',
      );

      const jsonData = JSON.parse(fs.readFileSync(jsonFilePath, 'utf-8'));
      console.log(jsonData);

      const workbook = new ExcelJS.Workbook();
      const excel = workbook.addWorksheet('Excel Data');

      excel.columns = [
        { header: 'InscriptionId', key: 'inscriptionId' },
        { header: 'ImageUrl', key: 'imageUrl' },
        { header: 'Name', key: 'name' },
        { header: 'Number', key: 'number' },
        { header: 'Hat', key: 'hat' },
        { header: 'Eye', key: 'eye' },
        { header: 'Mouth', key: 'mouth' },
        { header: 'Necklace', key: 'necklace' },
        { header: 'Shoes', key: 'shoes' },
      ];

      jsonData.forEach((item: any) => {
        let attributes: any = {};

        if (item.attributes && item.attributes.length > 0) {
          attributes = item.attributes[0];
        } else if (item.attributes) {
          attributes = item.attributes;
        }

        excel.addRow({
          inscriptionId: item.inscriptionId,
          imageUrl: item.imageUrl,
          name: item.name,
          number: item.number,
          hat: attributes?.Hat,
          eye: attributes?.Eye,
          mouth: attributes?.Mouth,
          necklace: attributes?.Necklace,
          shoes: attributes?.Shoes,
        });
      });

      const filePath = path.join(
        __dirname,
        '..',
        '..',
        '..',
        'data-excel.xlsx',
      );
      await workbook.xlsx.writeFile(filePath);
    } catch (err) {
      console.error('Error:', err);
      return runMeErrorHelper(err);
    }
  }
}
```

### Building the queue service

**N/B**: They're trade offs to these implementation. Having an isolated system for handling queue processing would be a preferred scalable solution. This takes the load off the server receiving requests. The system processes the data based on the priority, or whichever comes first. I am sharing this for the sole purpose of having to get things done quickly, with no other incurred cost like so. This is not the best implementation out there.

Typical systems used by big tech includes:

- [RabbitMq](https://www.rabbitmq.com/docs)
- [Apache Kafka](https://kafka.apache.org/documentation/)
- [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs)
- [Amazon MQ](https://aws.amazon.com/amazon-mq/) others includes [aws sns](https://aws.amazon.com/sns/) and [aws sqs](https://aws.amazon.com/sqs/)
- [Redis](https://redis.io/docs/latest/)

There are two separate approaches i'd talk about in this post. The baseline logic emulates how a queue operates `FIFO: First In First Out`, or selection based on level of priority etc.  
The second one, utilizes postgres as our database engine. The database handle retrieval of the data, and process it for queuing operation.  

### 1. Using basic queue data structure

```ts

@Injectable()
export class QueueService {
  private queue: string[] = [];

  private readonly logger = new Logger(QueueService.name);
  constructor() {}

    // add data to queue
  enqueue(id: string): void {
    this.queue.push(id);
    this.logger.log(`adding new id to the queue: ${id}`);
  }

    // remove data from the queue
  dequeue(): string | null {
    if (this.isEmpty()) {
      this.logger.log(
        `oops, the queue is empty cannot perform dequeue operation`,
      );
      return null;
    }

    const id = this.queue.shift();
    this.logger.log(`removing id from the queue: ${id}`);
    return id;
  }

  isEmpty(): boolean {
    return this.queue.length === 0;
  }

  getSize(): number {
    this.logger.log(`fetching the total size of the queue`);
    return this.queue.length;
  }
}

```

with this, we can run the operation based on the number of batches or data needed to be queried or so.

### 2. Using Postgres with SKIP LOCKED

> postgres, I use postgres for everything. - `unknown author`

![from twitter](/images/queue/twitter_postgres.jpg "image from twitter")

postgres can handle if not all, of the weight be thrown at. The image above, affirms the many reason why one should choose postgres. Although, not entirely true are they could be pitfalls. Some application are well suited for certain use cases.

I have used postgres for async queue processing, and it delivered just well for my needs.

#### How SKIP LOCKED works under the hood

Skip locked is a feature added in postgres 9.5 mainly for handling async queue processing to solve locking issues related with relational databases, for a guaranteed safety between transactions.

some advantages includes:

- improved performance
- efficiency for atomic commits
- avoid deadlocks
- acid gracefully

when not to use skip locked:

- ordering is important (could be negligible if data has a created date tag)
- when all rows must be processed
- long running transactions
- when retry mechanism is not considered

```sql
BEGIN;

SELECT * FROM orders
WHERE processed = false
FOR UPDATE SKIP LOCKED
LIMIT 20;

COMMIT;
```

- starts a transaction with the BEGIN statement
- selects an order where processed == false
- FOR UPDATE skips any row that is currently in use by another transaction or query
- LIMIT 20 limits to processing 20 rows per transaction
- COMMIT the transaction after

adding retry mechanism or batch processing

### dependencies

- `typeorm` orm for node  
- `pg` pg instance for node
- `nestjs` framework interfaced with express or fastify

```ts
@Injectable
export class QueueService{
    constructor(private queueRepository: Repository<Queue>, 
    private dataSource: DataSource
    ){}
    async handleBatchProcessing({ productId }: { productId: string }) {
    try {
      const getTotalProduct = await this.queueRepository.count({
        where: {
          product: {
            processed: false,
          },
        },
      });
      let processedCount: number = 0;
      let currentBatch: number = 0;
      while (processedCount < getTotalProduct) {
        const processedNumber =
          await this.queueOrderAndProcess(productId);
        processedCount += processedNumber;
        currentBatch++;
        this.logger.log(
          `Batch ${currentBatch} processed ${processedNumber} notifications.`,
        );
      }
    } catch (err) {
      throw err;
    }
  }

  private async queueOrderAndProcess(id: string): Promise<number>{
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try{
        // processed the retrieved order from skip locked (20) per transaction
        for (let i = 0; i < getOrder.length; i++) {
        const p = getOrder[i];
        this.logger.log(
          `inserting into queue table, ${getOrder.length - i - 1} left`,
        );
        // do something to your order that is received. send notifications or something

      }
      return getOrder.length;
    }catch(err){
        await queryRunner.rollbackTransaction();
        throw err;
    }finally{
        await queryRunner.release();
    }
  }
}
```

## TLDR

- [How to Choose a Message Queue](https://blog.bytebytego.com/p/how-to-choose-a-message-queue-kafka)
- [Skip Locked Queue Implementation with Postgres](https://www.2ndquadrant.com/en/blog/what-is-select-skip-locked-for-in-postgresql-9-5/)
- [pg-boss for queueing jobs in nodejs](https://github.com/timgit/pg-boss)
- [webscraping with puppeteer](https://apify.com/?fpr=ya6qx)
