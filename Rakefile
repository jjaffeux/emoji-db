require "nokogiri"
require "open-uri"
require "json"
require "base64"
require "fileutils"
require 'hashdiff'

CURRENT_EMOJI_LIST = "https://unicode.org/emoji/charts/full-emoji-list.html"
CURRENT_TONEABLE_EMOJI_LIST = "https://www.unicode.org/emoji/charts/full-emoji-modifiers.html"
EMOJI_ORDERING_LIST = "https://unicode.org/emoji/charts/emoji-ordering.html"
EMOJI_SEARCH_ALIASES_LIST = "https://raw.githubusercontent.com/unicode-org/cldr/main/common/annotations/en.xml"

TASKS = [
  {
    :type => :base,
    :url => CURRENT_EMOJI_LIST,
    :platform_cells => {
      :apple => 3,
      :google => 4,
      :twitter => 7,
      :emoji_one => 8,
      :facebook => 5,
      :samsung => 9,
      :windows => 6
    }
  },
  {
    :type => :tones,
    :url => CURRENT_TONEABLE_EMOJI_LIST,
    :platform_cells => {
      :apple => 3,
      :google => 4,
      :twitter => 7,
      :emoji_one => 8,
      :facebook => 5,
      :samsung => 9,
      :windows => 6
    }
  }
]

FITZPATRICK_SCALE = [ "1f3fb", "1f3fc", "1f3fd", "1f3fe", "1f3ff" ]

def code_to_emoji(code)
  code
    .split("_")
    .map { |e| e.to_i(16) }
    .pack "U*"
end

def write_emoji(path, image, resize=false)
  open(path, "wb") { |file| file << image }
  `convert #{path} -fuzz 10% -fill none -draw "alpha 0,0 floodfill" -channel alpha -blur 0x1 -level 50x100% +channel -background transparent -flatten -resize 72x72 #{path}` if resize
  `pngout #{path} -s0`
  putc "."
ensure
  if File.exists?(path) && !File.size?(path)
    raise "Failed to write emoji: #{path}"
  end
end

def cell_to_img(cell)
  return unless img = cell.at_css("img")
  Base64.decode64(img["src"][/base64,(.+)$/, 1])
end

def cldr_short_name_to_discourse_name(name)
  name.text.delete_prefix("⊛ ").gsub(':', '_').gsub(/\s+/, "_").gsub(/\W/,'').gsub('__','_').downcase
end

def cldr_short_name_to_discourse_name_toneable(name)
  name.text.split(':').first.delete_prefix("⊛ ").gsub(/\s+/, "_").gsub(/\W/,'').gsub('__','_').downcase
end

task :update_db do
  base_db = JSON.parse(File.read("db.json"))
  old_base_db = base_db.clone
  diff_db = {}

  task = TASKS.first
  list = open(task[:url]).read

  doc = Nokogiri::HTML(list)
  table = doc.css("table")[0]
  table.css("tr").each do |row|
    cells = row.css("td")

    if cells.size == 5
      code = cells[1].at_css("a")["name"]
      emoji = code_to_emoji(code)
      unicode_name = cldr_short_name_to_discourse_name(cells[4])

      if name = base_db[emoji]
        diff_db[emoji] = {discourse: name, unicode: unicode_name} unless name == unicode_name
      else
        name = unicode_name
        next if name.empty?
        base_db[emoji] = name
      end
    elsif cells.size == 15
      code = cells[1].at_css("a")["name"]
      emoji = code_to_emoji(code)
      unicode_name = cldr_short_name_to_discourse_name(cells[14])

      if name = base_db[emoji]
        diff_db[emoji] = {discourse: name, unicode: unicode_name} unless name == unicode_name
      else
        name = cldr_short_name_to_discourse_name(cells[14])
        next if name.empty?
        base_db[emoji] = name
      end
    end
  end

  puts "Differences between Discourse and Unicode shortcodes:"
  puts JSON.pretty_generate(diff_db)
  puts "#{diff_db.size} differences"

  puts "Shortcodes shared by different emojis:"
  base_db.each_with_object({}) { |(k,v),base_db| (base_db[v] ||= []) << k }.
          select { |_,v| v.size > 1 }.
          flat_map(&:last).
          each_slice(2){|a, b| puts; print diff_db[a]; puts a; print diff_db[b]; puts b}

  puts "New Emojis in the database:"
  puts JSON.pretty_generate(Hashdiff.diff(old_base_db, base_db))
  puts "#{base_db.size - old_base_db.size} new emojis added to the base_db"
  File.write("db.json", JSON.pretty_generate(base_db))
end


task :update_emoji do
  base_db = JSON.parse(File.read("db.json"))

  task = TASKS.first
  list = open(task[:url]).read

  doc = Nokogiri::HTML(list)
  table = doc.css("table")[0]
  table.css("tr").each do |row|
    cells = row.css("td")

    if cells.size == 5
      code = cells[1].at_css("a")["name"]
      emoji = code_to_emoji(code)

      name = base_db[emoji]
      next if name.empty?
      image = cells[3].at_css('img:last-of-type')
      next unless image
      image = Base64.decode64(image["src"][/base64,(.+)$/, 1])

      task[:platform_cells].each do |style, index|
        style_path = File.join("generated", style.to_s)
        FileUtils.mkdir_p(style_path)
        write_emoji(File.join(style_path, "#{name}.png"), image, true)
      end
    elsif cells.size == 15
      code = cells[1].at_css("a")["name"]
      emoji = code_to_emoji(code)

      name = base_db[emoji]
      next if name.empty?

      task[:platform_cells].each do |style, index|
        image = cell_to_img(cells[index])
        next unless image
        style_path = File.join("generated", style.to_s)
        FileUtils.mkdir_p(style_path)
        write_emoji(File.join(style_path, "#{name}.png"), image)
      end
    end
  end

  task = TASKS.last
  list = open(task[:url]).read

  doc = Nokogiri::HTML(list)
  table = doc.css("table")[0]
  table.css("tr").each do |row|
    cells = row.css("td")

    if cells.size == 5
      next
    elsif cells.size == 15
      code = cells[1].at_css("a")["name"]
      emoji = code_to_emoji(code)

      unscaled_codes = code.split("_")
      unscaled_codes.delete_at(1)
      unscaled_code = unscaled_codes.join("_")
      begin
        scale_index = FITZPATRICK_SCALE.index(code.split("_")[1]) + 2
        unscaled_emoji = code_to_emoji(unscaled_code)

        name = base_db[unscaled_emoji] || cldr_short_name_to_discourse_name_toneable(cells[14])
        task[:platform_cells].each do |style, index|
          image = cell_to_img(cells[index])
          next unless image
          style_path = File.join("generated", style.to_s, name)
          FileUtils.mkdir_p(style_path)
          write_emoji(File.join(style_path, "#{scale_index}.png"), image)
        end
      rescue StandardError => e
        puts code
        pp e
      end
    end
  end


  emojis = {}
  base_db.each do |char, name|
    emojis[char] = {
      :name => name,
      :fitzpatrick_scale => File.directory?("generated/emoji_one/#{name}")
    }
  end

  current_category = nil
  list = open(CURRENT_EMOJI_LIST).read
  doc = Nokogiri::HTML(list)
  table = doc.css("table")[0]
  table.css("tr").each do |row|
    if heading = row.css("th.mediumhead")[0]
      current_category = heading.text
    end

    cells = row.css("td")
    next if cells.size != 15

    if emoji = emojis[cells[2].text]
      emoji[:category] = current_category
    end
  end

  translations = {
    ':)' => 'slight_smile',
    ':-)' => 'slight_smile',
    '^_^' => 'slight_smile',
    '^__^' => 'slight_smile',
    ':(' => 'frowning',
    ':-(' => 'frowning',
    ';)' => 'wink',
    ';-)' => 'wink',
    ":'(" => 'cry',
    ":'-(" => 'cry',
    ":-'(" => 'cry',
    ':p' => 'stuck_out_tongue',
    ':P' => 'stuck_out_tongue',
    ':-P' => 'stuck_out_tongue',
    ':O' => 'open_mouth',
    ':-O' => 'open_mouth',
    ':D' => 'smiley',
    ':-D' => 'smiley',
    ':|' => 'expressionless',
    ':-|' => 'expressionless',
    ':/' => 'confused',
    '8-)' => 'sunglasses',
    ';P' => 'stuck_out_tongue_winking_eye',
    ';-P' => 'stuck_out_tongue_winking_eye',
    ':$' => 'blush',
    ':-$' => 'blush'
  }

  sections = {}

  list = open(EMOJI_ORDERING_LIST).read
  doc = Nokogiri::HTML(list)
  table = doc.css("table")[0]
  section = nil
  table.css("tr th").each do |row|
    section_name = row.css("a").attr("name").value

    if row.attr('class') === 'bighead'
      section = section_name
      sections[section] = {
        fullname: row.css("a").text,
        sub_sections: []
      }
    else
      sections[section][:sub_sections] << section_name
    end
  end

  sections.delete("component")

  list = open(EMOJI_SEARCH_ALIASES_LIST).read
  doc = Nokogiri::XML(list)
  doc.xpath("//annotation").each do |node|
    emoji = node.attr("cp")
    next if emoji.nil? || emoji.strip.empty?
    json = emojis[emoji]
    next if json.nil?
    
    search_aliases = json[:search_aliases]
    search_aliases ||= []
    search_aliases += node.text.split("|").map(&:strip).reject { |a| a.gsub(" ", "_") == json[:name] }
    emojis[emoji][:search_aliases] = search_aliases.uniq
  end

  db_path = File.join("generated", "db.json")
  File.write(db_path, JSON.pretty_generate(emojis: emojis, translations: translations, sections: sections))
end
