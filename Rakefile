require "nokogiri"
require "open-uri"
require "json"
require "base64"
require "fileutils"

CURRENT_EMOJI_LIST = "https://unicode.org/emoji/charts-13.0/full-emoji-list.html"
EMOJI_ORDERING_LIST = "https://unicode.org/emoji/charts/emoji-ordering.html"

TASKS = [
  {
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
  }
]

FITZPATRICK_SCALE = [ "1f3fb", "1f3fc", "1f3fd", "1f3fe", "1f3ff" ]

def code_to_emoji(code)
  code
    .split("_")
    .map { |e| e.to_i(16) }
    .pack "U*"
end

def write_emoji(path, image)
  open(path, "wb") { |file| file << image }
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

task :default do
  base_db = JSON.parse(File.read("db.json"))

  TASKS.each do |task|
    list = open(task[:url]).read

    doc = Nokogiri::HTML(list)
    table = doc.css("table")[0]
    table.css("tr").each do |row|
      cells = row.css("td")

      if cells.size == 5
        code = cells[1].at_css("a")["name"]
        emoji = code_to_emoji(code)
        name = base_db[emoji]
        next unless name
        image = cells[3].at_css('img:last-of-type')
        next unless image
        image = Base64.decode64(image["src"][/base64,(.+)$/, 1])

        task[:platform_cells].each do |style, index|
          style_path = File.join("generated", style.to_s)
          FileUtils.mkdir_p(style_path)
          write_emoji(File.join(style_path, "#{name}.png"), image)
        end
      elsif cells.size == 15
        code = cells[1].at_css("a")["name"]
        emoji = code_to_emoji(code)

        if name = base_db[emoji]
          task[:platform_cells].each do |style, index|
            image = cell_to_img(cells[index])
            next unless image
            style_path = File.join("generated", style.to_s)
            FileUtils.mkdir_p(style_path)
            write_emoji(File.join(style_path, "#{name}.png"), image)
          end
        elsif FITZPATRICK_SCALE.any? { |scale| code[scale] }
          unscaled_codes = code.split("_")
          unscaled_codes.delete_at(1)
          unscaled_code = unscaled_codes.join("_")
          scale_index = FITZPATRICK_SCALE.index(code.split("_")[1]) + 2
          unscaled_emoji = code_to_emoji(unscaled_code)

          if name = base_db[unscaled_emoji]
            task[:platform_cells].each do |style, index|
              image = cell_to_img(cells[index])
              next unless image
              style_path = File.join("generated", style.to_s, name)
              FileUtils.mkdir_p(style_path)
              write_emoji(File.join(style_path, "#{scale_index}.png"), image)
            end
          end
        end
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

  db_path = File.join("generated", "db.json")
  File.write(db_path, JSON.pretty_generate(emojis: emojis, translations: translations, sections: sections))
end
