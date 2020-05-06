# webpy_spina_cms
the way to use webpy img with spina cms  
app/model/spina/text.rb can be like this.

```ruby:app/model/spina/text.rb
module Spina
  class Text < ApplicationRecord
    extend Mobility
    translates :content, fallbacks: true
    has_many :page_parts, as: :page_partable
    has_many :layout_parts, as: :layout_partable
    has_many :structure_parts, as: :structure_partable
    before_save :webpify

    def webpify
      doc = Nokogiri::HTML.parse(self.content)
      doc.xpath('//figure').each do |node|
        img_url = node.at_xpath('.//img')[:src]
        if img_url.include?('blobs/')
          blob_signed = img_url.split('blobs/')[1].split('/')[0]
          blob = ActiveStorage::Blob.find_signed(blob_signed)
          webp_url = blob.representation(define: 'webp:lossless=false', convert: 'webp', resize_to_limit: [ 800, 600 ]).processed.url
          resized_url = blob.representation(quality: 55, resize_to_limit: [ 800, 600 ]).processed.url
          inner_html = ApplicationController.render(
            template: 'active_storage/blobs/_spina_blob',
            layout: false,
            locals: { blob: blob }
          )
          node.attributes['data-trix-attachment'].value = ApplicationController.render json: JSON.generate({ content: inner_html })
          node.replace inner_html
        end
      end
      self.content = doc.to_html
    end

    def webpify!
      webpify
      self.save!
    end
  end
end
```
  
active_storage/blobs/_spina_blob.html.erb can be like this.  

```erb:_spina_blob.html.erb
<% if blob.representable? %>
  <picture>
    <source type="image/webp" srcset="<%= blob.representation(define: 'webp:lossless=false', convert: 'webp', resize_to_limit: [ 800, 600 ]).processed.url %>">
    <img src="<%= blob.representation(quality: 55, resize_to_limit: [ 800, 600 ]).processed.url %>">
  </picture>
<% end %>

<figcaption class="attachment__caption">
  <% if caption = blob.try(:caption) %>
    <%= caption %>
  <% end %>
</figcaption>
```
