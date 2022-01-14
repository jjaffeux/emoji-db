# Emoji DB

### How to update

#### Emoji DB

1. Run `rake update_db` and fix any problems.

2. Run  `rake update_emoji`

#### Discourse

1. Run `rake emoji:update` 

2. Bump the value in https://github.com/discourse/discourse/blob/main/app/models/emoji.rb#L5

3. Run `rake javascript:update_constants`