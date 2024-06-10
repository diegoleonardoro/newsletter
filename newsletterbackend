import { Db } from 'mongodb';
import { connectToDatabase } from '../index';
import { BadRequestError } from '../../errors/bad-request-error';
import { sendNewsLetterEmail } from '../../services/emailVerification';
import sgMail from '@sendgrid/mail';
import { ObjectId } from 'mongodb';
import { v4 as uuidv4 } from 'uuid'; // For generating UUIDs
import { Document } from 'mongodb';
import { createClient, RedisClientType } from 'redis';
import { gunzip } from 'zlib';
import { promisify } from 'util';
import { zipcodesCommunityBoards } from "./zipcodes_communityboards"
import { articlesNewsLetter } from "./newsLetterArticles"

interface Subscriber {
  _id: ObjectId;
  email: string;
  name?: string;
  newsletter: boolean;
  frequency?: string;
  lastSent?: Date;
  identifier: string;
  zipcode?: string;
}

interface Data311Call {
  'Created Date': string;
  Agency: string;
  'Agency Name': string;
  'Complaint Type': string;
  Descriptor: string;
  'Location Type': string;
  'Incident Zip': string;
  'Incident Address': string;
  Borough: string;
  Location: {
    latitude: string;
    longitude: string;
    human_address: string;
  };
}

interface DOBPermit {
  'House Number': string;
  'Street Name': string;
  'Borough': string;
  'Community Board': string;
  'Job Description': string;
  'Estimated Cost': string;
  'Issued Date': string;
  'Expired Date': string
}

interface RedisDataResponse {
  message?: string;
  statusCode?: number;
  data?: any;
}

interface ProcessedData {
  subscriber: string;
  name: string;
  calls: Data311Call[];
}


export class NewsletterRepository {

  private db: Promise<Db>;
  private collectionName = 'newsletter';
  private cacheKey311Calls = "complaints_data";
  private redisClient: RedisClientType;


  constructor() {
    this.db = connectToDatabase();
    this.redisClient = createClient({
      url: `redis://${process.env.REDIS_HOST}:${process.env.REDIS_PORT}`
    })
    this.redisClient.connect().then(() => {
      console.log('Successfully connected to Redis');
    })
      .catch(error => {
        console.error('Failed to connect to Redis:', error);
      });
  };

  private async fetchSubscribers() {

    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);
    const now = new Date();
    const query = {
      $or: [
        { lastSent: { $exists: false } },
        {
          $expr: {
            $gte: [
              { $subtract: [now, "$lastSent"] },
              { $multiply: [{ $toDecimal: "$frequency" }, 518400000] } // Ensure frequency is a number
            ]
          }
        }
      ]
    };

    const newslettersToSend = await emailsCollection.find(query).toArray();
    return newslettersToSend;
  };

  private async fetchAllSubscribers() {
    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);
    const users = await emailsCollection.find({ email: "diegoinbox0@gmail.com"}).toArray();
    return users;
  }

  private async sendEmails(subscribers: any[]) {

    sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

    if (subscribers.length === 0) {
      return { message: "No subscribers.", statusCode: 404 };
    }
    const msg = {
      from: { name: "Insider Hood", email: "admin@insiderhood.com" },
      personalizations: subscribers.map(subscriber => ({
        to: [{ email: subscriber.email }],
        dynamic_template_data: {
          name: subscriber.name,
          //   unsubscribeButton: `<a href="${process.env.BASE_URL?.split(" ")[0]}/newsletterpreferences?user_id=${subscriber.identifier}"
          //   style="background-color: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 5px;">
          //   Click Here
          // </a>`
        }
      })),
      templateId: "d-a9c9d7ddcb51448e8c73e2e26f25b14d"
    };
    try {
      await sgMail.send(msg);
      const ids = subscribers.map(sub => sub._id);
      await this.updateLastSent(ids);
      return { message: "Newsletter Sent", statusCode: 200 };
    } catch (error) {
      console.error("Error sending email: ", error);
      return { message: "Failed to send newsletter", statusCode: 400 };
    }

  }

  private async updateLastSent(ids: string[]) {


    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);
    // Convert string array to ObjectId array
    const objectIds = ids.map(id => new ObjectId(id));
    await emailsCollection.updateMany(
      { _id: { $in: objectIds } },
      { $set: { lastSent: new Date() } }
    );
  }

  private getOrdinal(n: number) {
    const s = ["th", "st", "nd", "rd"];
    const v = n % 100;
    return n + (s[(v - 20) % 10] || s[v] || s[0]);
  }

  private formatCallToHTML(call: Data311Call) {

    const date = new Date(call['Created Date']);
    const day = date.getUTCDate();

    // Separate the day number and suffix concatenation
    const dayWithOrdinal = this.getOrdinal(day);
    const formattedDate = date.toLocaleDateString('en-US', {
      month: 'long',
      day: 'numeric',
      timeZone: 'UTC'
    }).replace(day.toString(), dayWithOrdinal);  // Replace the numeric day with the day with its ordinal suffix

    return `
    <div style="border: 1px solid #ccc; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); background-color: #ffffff; margin-bottom: 12px; padding: 20px; font-family: 'Helvetica', 'Arial', sans-serif;">
      <div style="background-color: #f2f4f8; padding: 10px; margin: -20px -20px 20px -20px; border-radius: 10px 10px 0 0;">
        <h3 style="color: #333; font-size: 18px; font-weight: bold;">${call['Complaint Type']}</h3>
      </div>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Date:</strong> <span style="color: #555;">${formattedDate}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Agency:</strong> <span style="color: #555;">${call['Agency']}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Descriptor:</strong> <span style="color: #555;">${call['Descriptor']}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Location Type:</strong> <span style="color: #555;">${call['Location Type']}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Zip:</strong> <span style="color: #555;">${call['Incident Zip']}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Address:</strong> <span style="color: #555;">${call['Incident Address']}</span></p>
      <p style="margin: 5px 0; font-size: 16px;"><strong>Borough:</strong> <span style="color: #555;">${call['Borough']}</span></p>
    </div>
  `;
  }

  private formatPermitToHTML(permit: DOBPermit): string {

    const formatDOBDate = (dateString: string): string => {
      const date = new Date(dateString);
      const day = date.getUTCDate();
      const ordinalDay = this.getOrdinal(day);
      const formattedDate = date.toLocaleDateString('en-US', {
        month: 'long',
        day: 'numeric',
        timeZone: 'UTC'
      });
      return formattedDate.replace(day.toString(), ordinalDay);  // Correctly replace the day with the day and its ordinal suffix
    }

    const estimatedCost = new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 2
    }).format(Number(permit['Estimated Cost']));

    const borough = permit.Borough.charAt(0).toUpperCase() + permit.Borough.slice(1).toLowerCase();

    return `
      <div style="border: 1px solid #ccc; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); background-color: #ffffff; margin-bottom: 12px; padding: 20px; font-family: 'Helvetica', 'Arial', sans-serif; text-align: left;">
        <div style="background-color: #f2f4f8; padding: 10px; margin: -20px -20px 20px -20px; border-radius: 10px 10px 0 0; text-align: left;">
          <h3 style="color: #333; font-size: 18px; font-weight: bold; text-align: left;">${permit['House Number']} ${permit['Street Name']}</h3>
        </div>
        <p style="margin: 5px 0; font-size: 16px; text-align: left;"><strong>Borough:</strong> <span style="color: #555;">${borough}</span></p>
        <p style="margin: 5px 0; font-size: 16px; text-align: left;"><strong>Job Description:</strong> <span style="color: #555;">${permit['Job Description']}</span></p>
        <p style="margin: 5px 0; font-size: 16px; text-align: left;"><strong>Issued Date:</strong> <span style="color: #555;">${formatDOBDate(permit['Issued Date'])}</span></p>
        <p style="margin: 5px 0; font-size: 16px; text-align: left;"><strong>Expired Date:</strong> <span style="color: #555;">${formatDOBDate(permit['Expired Date'])}</span></p>
        <p style="margin: 5px 0; font-size: 16px; text-align: left;"><strong>Estimated Cost:</strong> <span style="color: #555;">${estimatedCost}</span></p>
      </div>
      `;
  }

  private async fetchDataFromRedis(key: string): Promise<RedisDataResponse> {
    const base64Data = await this.redisClient.get(key);
    if (!base64Data) {
      console.log(`No data found for ${key}.`);
      return { message: `No data found for ${key}`, statusCode: 404 };
    }

    const buffer = Buffer.from(base64Data, 'base64');
    const decompressAsync = promisify(gunzip);
    try {
      const decompressedData = await decompressAsync(buffer);
      return { data: JSON.parse(decompressedData.toString('utf-8')) };
    } catch (error) {
      console.error(`Error processing data for ${key}: ${error}`);
      return { message: `Error processing data for ${key}`, statusCode: 500 };
    }
  }

  private create311CallsButtonHTML(zipCodes: string[]): string {

    const baseUrlForEmailVerification = process.env.BASE_URL ?process.env.BASE_URL.split(" ")[0] : ''
    const zipCodeParam = encodeURIComponent(zipCodes.join(','));
    
    return `
    <div style="text-align: center; margin-top: 20px;">
      <a href="https://insiderhood.com/311complaints?zips=${zipCodeParam}" style="background-color: #000000; color: white; padding: 14px 20px; margin: 10px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; display: inline-block;">See More 311 Calls</a>
    </div>
  `;
  }
////${baseUrlForEmailVerification}
  private createDOBPermitsButtonHTML(communityBoards: string[]): string {
    const baseUrlForEmailVerification = process.env.BASE_URL ? process.env.BASE_URL.split(" ")[0] : ''
    const communityBoardParam = encodeURIComponent(communityBoards.join(','));
    return `
    <div style="text-align: center; margin-top: 20px;">
      <a href="https://insiderhood.com/DOBApprovedPermits?cb=${communityBoardParam}" style="background-color: #000000; color: white; padding: 14px 20px; margin: 10px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; display: inline-block;">See More DOB Approved Permits</a>
    </div>
  `;
  }

  async subscribeToNewsletter(data: { email: string, name?: string, newsletter: boolean, frequency?: string, zipCode?: string }): Promise<{ message: string }> {

    const db = await this.db;
    const { email, name = null, newsletter, zipCode, frequency = 'everyweek' } = data;
    const emailsCollection = db.collection(this.collectionName);
    const existingEmail = await emailsCollection.findOne({ email });

    if (existingEmail) {
      throw new BadRequestError("Email in use");
    };

    const newData = { email, name, newsletter, frequency, zipCode };
    await emailsCollection.insertOne(newData);

    sendNewsLetterEmail({
      email: email,
      baseUrlForEmailVerification: process.env.BASE_URL ? process.env.BASE_URL.split(" ")[0] : ''
    });
    return { message: "Email subscribed" };

  };

  async sendNewsLetter(): Promise<{ message: string, statusCode: number }> {

    try {
      const subscribers = await this.fetchSubscribers();//data.frequency
      return ({ message: '', statusCode: 2 });
      // return await this.sendEmails(subscribers);

    } catch (error) {
      console.error("Error processing newsletter:", error);
      return { message: "Failed to send newsletter.", statusCode: 500 };
    }

  };

  async sendReferralEmail(data: { email: string, templateId: string }): Promise<{ message: string, statusCode: number }> {

    sgMail.setApiKey(process.env.SENDGRID_API_KEY!);
    const { email, templateId } = data;
    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);
    const existingEmail = await emailsCollection.findOne({ email });
    if (existingEmail) {
      throw new BadRequestError("Email in use");
    };

    try {
      const msg = {
        to: email,
        from: {
          name: "Insider Hood",
          email: "admin@insiderhood.com"
        },
        templateId: templateId
      };
      await sgMail.send(msg);
      return { message: "Newsletter Sent", statusCode: 200 };

    } catch (error) {
      console.error("Error retrieving subscribers:", error); // Catch and log any errors during the query
      return { message: `Failed to send newsletter.`, statusCode: 500 }
    }

  }

  async updateUsers(data: { identifier?: string, updates?: {} }): Promise<{ message: string, statusCode: number }> {
    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);

    if (!data.identifier) {
      const allUsersCursor = emailsCollection.find({});
      const allUsers = await allUsersCursor.toArray();

      for (const doc of allUsers) {
        const updateResult = await emailsCollection.updateOne(
          { _id: doc._id },
          { $set: { identifier: uuidv4() } }
        );
        console.log(updateResult);
      }
      return { message: `Updated documents with UUIDs`, statusCode: 200 };
    } else {
      // Update specific document by identifier
      const updateResult = await emailsCollection.updateOne(
        { identifier: data.identifier },
        { $set: data.updates }
      );

      if (updateResult.matchedCount === 0) {
        return { message: 'No document found with the provided identifier', statusCode: 200 };
      } else {
        return { message: 'Document updated successfully', statusCode: 200 };
      }
    }
  };


  async geoBasedNewsLetter(): Promise<{ message: string, statusCode: number }> {

    try {


      // --- --- retreive 311 calls and dob permits from redis --- --- 
      const [data311Calls, dobPermits] = await Promise.all([
        this.fetchDataFromRedis('complaints_data'),
        this.fetchDataFromRedis('dob_approved_permits')
      ]);

      // Handle errors after both calls
      if (data311Calls.statusCode) {
        console.error('Failed to fetch 311 call data:', data311Calls.message);
        return { message: `Failed to fetch 311 call data: ${data311Calls.message}`, statusCode: data311Calls.statusCode };
      }

      if (dobPermits.statusCode) {
        console.error('Failed to fetch DOB permits data:', dobPermits.message);
        return { message: `Failed to fetch DOB permits data: ${dobPermits.message}`, statusCode: dobPermits.statusCode };
      }
      // --- --- --- --- --- --- --- --- --- 


      // --- ---  retreive users from the users db --- ---
      const subscribers = await this.fetchAllSubscribers();
      // --- --- --- --- --- --- --- --- --- 


      // --- --- filter data311Calls according to the subscriber.zipcode
      const formattedResults = subscribers.map(subscriber => {

        const getCurrentDate = () => {
          const date = new Date();
          return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' }).toUpperCase();
        };

        const dayOfWeek = () => {
          const date = new Date();
          const weekday = date.toLocaleDateString('en-US', { weekday: 'long' });
          return weekday.charAt(0).toUpperCase() + weekday.slice(1).toLowerCase();
        }

        // Map zip codes to community boards
        const zipToBoard: Record<string, string> = Object.fromEntries(zipcodesCommunityBoards.map(obj => [Object.values(obj)[0], Object.keys(obj)[0]]));

        if (!subscriber.zipcodes || subscriber.zipcodes.length === 0) {
          return {
            to: [{ email: subscriber.email }],
            dynamic_template_data: {
              name: subscriber.name,
              complaintsHtml: "No relevant data available.", // Or handle this scenario differently
              currentDate: getCurrentDate(),
              day: dayOfWeek(),
              articlesNewsLetter: articlesNewsLetter.join("")
            }
          };
        }


        // Filter DOB approved permits based on community board
        const communityBoards = subscriber.zipcodes.map((zip: string) => zipToBoard[zip]).filter(Boolean);

        const relevantDOBPermits = dobPermits.data
          .filter((permit: DOBPermit) => communityBoards.includes(permit['Community Board']))
          .slice(0, 4)
          .map((permit: DOBPermit) => this.formatPermitToHTML(permit))
          .join('');

        const permitsButtonHTML = this.createDOBPermitsButtonHTML(communityBoards);
       
        // --- --- --- --- --- --- --- --- --- --- 



        // filter 311 calls:
        const relevant311Calls = data311Calls.data
          .filter((call: Data311Call) => subscriber.zipcodes.includes(call['Incident Zip']))
          .slice(0, 4)
          .map((call: Data311Call) => this.formatCallToHTML(call))
          .join('');

        const calls311ButtonHTML = this.create311CallsButtonHTML(subscriber.zipcodes);
        
        // --- --- --- --- --- --- --- --- --- --- 


        return {
          to: [{ email: subscriber.email }],
          dynamic_template_data: {
            name: subscriber.name,
            complaintsHtml: relevant311Calls,
            relevantDOBPermits: relevantDOBPermits,
            currentDate: getCurrentDate(),
            day: dayOfWeek(),
            calls311ButtonHTML: calls311ButtonHTML,
            permitsButtonHTML: permitsButtonHTML,
            articlesNewsLetter: articlesNewsLetter.join("")
          }
        };


      });
      // --- --- --- --- --- --- --- --- ---



      const msg = {
        from: { name: "Insider Hood", email: "admin@insiderhood.com" },
        personalizations: formattedResults,
        templateId: "d-6eb53e9e9d8a40e4bfa8150ec791cb7b"
      };

      sgMail.setApiKey(process.env.SENDGRID_API_KEY!);
      await sgMail.send(msg);

      return { message: 'Newsletter Sent', statusCode: 200 };


    } catch (error) {
      console.error('An error occurred:', error);
      return { message: 'Error during data retrieval and processing', statusCode: 500 };
    }
  }


  async getUserInfo(data: { identifier: string }): Promise<{ user: Document, statusCode: number }> {
    const db = await this.db;
    const emailsCollection = db.collection(this.collectionName);
    const { identifier } = data;
    const existingUser = await emailsCollection.findOne({ identifier });
    if (!existingUser) {
      throw new BadRequestError("No user found");
    };
    return ({ user: existingUser, statusCode: 200 })
  }


}
