# strapi-provider-upload-aws-s3-custom

## Resources

- [LICENSE](LICENSE)

## Configuration

- `provider` defines the name of the provider
- `providerOptions` is passed down during the construction of the provider. (ex: `new AWS.S3(config)`). [Complete list of options](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#constructor-property)
- `providerOptions.params` is passed directly to the parameters to each method respectively.
  - `ACL` is the access control list for the object. Defaults to `public-read`.
  - `signedUrlExpires` is the number of seconds before a signed URL expires. (See [how signed URLs work](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-signed-urls.html)). Defaults to 15 minutes and URLs are only signed when ACL is set to `private`.
  - `Bucket` is the name of the bucket to upload to.
- `actionOptions` is passed directly to the parameters to each method respectively. You can find the complete list of [upload/ uploadStream options](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#upload-property) and [delete options](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#deleteObject-property)

See the [documentation about using a provider](https://docs.strapi.io/developer-docs/latest/plugins/upload.html#using-a-provider) for information on installing and using a provider. To understand how environment variables are used in Strapi, please refer to the [documentation about environment variables](https://docs.strapi.io/developer-docs/latest/setup-deployment-guides/configurations/optional/environment.html#environment-variables).

If you're using the bucket as a CDN and deliver the content on a custom domain, you can get use of the `baseUrl` and `rootPath` properties to configure how your assets' urls will be saved inside Strapi.

### Provider Configuration

`./config/plugins.js` or `./config/plugins.ts` for TypeScript projects:

```js
module.exports = ({ env }) => ({
  // ...
  upload: {
    config: {
      provider: 'strapi-provider-upload-aws-s3-custom',
      providerOptions: {
        baseUrl: env('CDN_URL'),
        rootPath: env('CDN_ROOT_PATH'),
        s3Options: {
          accessKeyId: env('AWS_ACCESS_KEY_ID'),
          secretAccessKey: env('AWS_ACCESS_SECRET'),
          region: env('AWS_REGION'),
          params: {
            ACL: env('AWS_ACL', 'public-read'),
            signedUrlExpires: env('AWS_SIGNED_URL_EXPIRES', 15 * 60),
            Bucket: env('AWS_BUCKET'),
          },
        },
      },
      actionOptions: {
        upload: {},
        uploadStream: {},
        delete: {},
      },
    },
  },
  // ...
});
```

### Configuration for a private S3 bucket and signed URLs

If your bucket is configured to be private, you will need to set the `ACL` option to `private` in the `params` object. This will ensure file URLs are signed.

**Note:** If you are using a CDN, the URLs will not be signed.

You can also define the expiration time of the signed URL by setting the `signedUrlExpires` option in the `params` object. The default value is 15 minutes.

`./config/plugins.js`

```js
module.exports = ({ env }) => ({
  // ...
  upload: {
    config: {
      provider: 'strapi-provider-upload-aws-s3-custom',
      providerOptions: {
        accessKeyId: env('AWS_ACCESS_KEY_ID'),
        secretAccessKey: env('AWS_ACCESS_SECRET'),
        region: env('AWS_REGION'),
        params: {
          ACL: 'private', // <== set ACL to private
          signedUrlExpires: env('AWS_SIGNED_URL_EXPIRES', 15 * 60),
          Bucket: env('AWS_BUCKET'),
        },
      },
      actionOptions: {
        upload: {},
        uploadStream: {},
        delete: {},
      },
    },
  },
  // ...
});
```

#### Configuration for S3 compatible services

This plugin may work with S3 compatible services by using the `endpoint` option instead of `region`. Scaleway example:
`./config/plugins.js`

```js
module.exports = ({ env }) => ({
  // ...
  upload: {
    config: {
      provider: 'strapi-provider-upload-aws-s3-custom',
      providerOptions: {
        accessKeyId: env('SCALEWAY_ACCESS_KEY_ID'),
        secretAccessKey: env('SCALEWAY_ACCESS_SECRET'),
        endpoint: env('SCALEWAY_ENDPOINT'), // e.g. "s3.fr-par.scw.cloud"
        params: {
          Bucket: env('SCALEWAY_BUCKET'),
        },
      },
    },
  },
  // ...
});
```

### Security Middleware Configuration

Due to the default settings in the Strapi Security Middleware you will need to modify the `contentSecurityPolicy` settings to properly see thumbnail previews in the Media Library. You should replace `strapi::security` string with the object bellow instead as explained in the [middleware configuration](https://docs.strapi.io/developer-docs/latest/setup-deployment-guides/configurations/required/middlewares.html#loading-order) documentation.

`./config/middlewares.js`

```js
module.exports = [
  // ...
  {
    name: 'strapi::security',
    config: {
      contentSecurityPolicy: {
        useDefaults: true,
        directives: {
          'connect-src': ["'self'", 'https:'],
          'img-src': [
            "'self'",
            'data:',
            'blob:',
            'market-assets.strapi.io',
            'yourBucketName.s3.yourRegion.amazonaws.com',
          ],
          'media-src': [
            "'self'",
            'data:',
            'blob:',
            'market-assets.strapi.io',
            'yourBucketName.s3.yourRegion.amazonaws.com',
          ],
          upgradeInsecureRequests: null,
        },
      },
    },
  },
  // ...
];
```

If you use dots in your bucket name, the url of the ressource is in directory style (`s3.yourRegion.amazonaws.com/your.bucket.name/image.jpg`) instead of `yourBucketName.s3.yourRegion.amazonaws.com/image.jpg`. Then only add `s3.yourRegion.amazonaws.com` to img-src and media-src directives.

## Bucket CORS Configuration

If you are planning on uploading content like GIFs and videos to your S3 bucket, you will want to edit its CORS configuration so that thumbnails are properly shown in Strapi. To do so, open your Bucket on the AWS console and locate the _Cross-origin resource sharing (CORS)_ field under the _Permissions_ tab, then amend the policies by writing your own JSON configuration, or copying and pasting the following one:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET"],
    "AllowedOrigins": ["YOUR STRAPI URL"],
    "ExposeHeaders": [],
    "MaxAgeSeconds": 3000
  }
]
```

## Required AWS Policy Actions

These are the minimum amount of permissions needed for this provider to work.

```json
"Action": [
  "s3:PutObject",
  "s3:GetObject",
  "s3:ListBucket",
  "s3:DeleteObject",
  "s3:PutObjectAcl"
],
```
