[Facebook Marketing API Docs](https://developers.facebook.com/docs/reference/ads-api/)

#### Objects

- Business (Tophatter [760977220612233])
- Account (Marketing API [1051938118182807], Tophatter [861827983860489])
- Campaign (Android App Installs [6060101423257], Facebook App Installs [6060102414657])
- Ad Set
- Ad
- Ad Creative (Only via API)
- Ad Image (Only via API)

#### Associations

- Business has many Accounts.
- Account has many Campaigns.
- Campaign has many Ad Sets.
- Ad Set has many Ads.
- Ad has one Ad Creative (API only).
- Ad Creative has many Ad Images (API only).

#### Object Details

A Campaign has a particular objective:
- Website conversions OR
- App installs.

An Ad Set has an audience, budget, and schedule.
- For Android app installs we require the following:
  * OS Version 4.0+
  * Android Smartphones (all) + Android Tablets (all)
  * All mobile users
- For iOS app installs we require the following:

An Ad can have two formats:
- A single image.
- Multiple images (carousel).
- For mobile app installs an ad has the following:
  * Images (The minimum dimensions are 600x600)
  * Headline (Example: "Up To 80% Off")
  * Text (Example: "Lowest Prices + Free Shipping on select items.")
  * CTA ("Install Now" is the default)

#### Tophatter User Attribution

- source (Which ad network did the user come from? - "Facebook Ads", "Google" - This is event.media_source in AppsFlyer)
- ad_group (Which specific ad did the user come from? - "Tops", "Watches" - This is event.fb_adgroup_name in AppsFlyer)
- ad_campaign (Which specific ad set did the user come from? - "" - This is event.fb_adset_name in AppsFlyer)
- ad_category # Not used yet.
- ad_product_id # Not used yet.

#### Usage - Ads Management

```
ad_account = Zuck::AdAccount.find(1051938118182807)
campaigns = ad_account.campaigns
```

#### Usage - Audience Management

TBD

#### Usage - Ads Insights

TBD

#### Finding Creatives

Top 25 most-sold tops in the last 30 days:

```
product_ids = Invoice.where(paid_at: 1.month.ago..Time.now).joins(:lot).where('lots.product_category': 'Apparel').where('lots.product_subcategory': 'Tops').group('lots.product_id').count.sort_by { |k, v| v }.last(25).map(&:first)
lot_ids = product_ids.collect { |product_id| Lot.find_by!(product_id: product_id).id }
image_urls = product_ids.collect { |product_id| Lot.find_by!(product_id: product_id).image1.url(:original) }
image_urls.each do |image_url|
  pathname = Pathname.new(image_url)
  md5 = pathname.dirname.basename
  File.open("/Users/cte/Dropbox/Ads/#{md5}.jpg", "wb") do |f|
    f.binmode
    print "GET #{image_url}... "
    f.write HTTParty.get(image_url).parsed_response
    puts "Done"
    f.close
  end
end
```

Ash's approach:

```
def creatives(subcategory, count: 100, range: 2.weeks.ago..Time.now, min_width: 600, min_height: 600)
  lot_ids          = Lot.where(bidding_ended_at: range, product_subcategory: subcategory).pluck(:id)
  bidders          = Bid.joins(lot: :image1, user: {}).where('DATEDIFF(bids.created_at, users.created_at) = 0').where(lot_id: lot_ids, 'images.width': min_width..Float::INFINITY, 'images.height': min_height..Float::INFINITY).group('images.data_fingerprint').count('distinct bids.user_id')
  fingerprints     = bidders.sort_by { |f, v| -v }.first(count).collect(&:first)
  latest_image_ids = Image.where(data_fingerprint: fingerprints).group(:data_fingerprint).maximum(:id).values​
  Image.where(id: latest_image_ids).sort_by { |i| fingerprints.index(i.data_fingerprint) }.collect { |i| i.url(:large) }
end
```
