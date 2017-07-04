require "nokogiri"
require "open-uri"
require "json"
require "base64"
require "fileutils"

CURRENT_EMOJI_LIST = "http://unicode.org/emoji/charts/full-emoji-list.html"

tasks = [
  {
    :url => CURRENT_EMOJI_LIST,
    :platform_cells => {
      :apple => 3,
      :google => 4,
      :twitter => 5,
      :emoji_one => 6,
      :facebook => 7,
      :facebook_messenger => 8,
      :samsung => 9,
      :windows => 10
    }
  },
  {
    :url => "http://unicode.org/emoji/charts-beta/full-emoji-list.html",
    :platform_cells => {
      :google_blob => 4
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

task "emoji_list" do
  base_db = JSON.parse(File.read("db.json"))

  tasks.each do |task|
    list = open(task[:url]).read

    doc = Nokogiri::HTML(list)
    table = doc.css("table")[0]
    table.css("tr").each do |row|
      cells = row.css("td")

      # skip header and section rows
      next if cells.size != 16

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
    next if cells.size != 16

    if emoji = emojis[cells[2].text]
      emoji[:category] = current_category
    end
  end

  db_path = File.join("generated", "db.json")
  File.write(db_path, JSON.pretty_generate(emojis))
end
