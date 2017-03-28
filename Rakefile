# frozen_string_literal: true
require 'bundler/setup'
Bundler.require

require 'json'
require 'pathname'
require 'tempfile'
require 'time'
require 'uri'
require 'zlib'

def expand(syntax)
  arrays = syntax.scan(/\w+|\[.*?\]|\S+/).map { |part|
    case part
    when /^\[.*\|/
      part.delete('[]').split('|').map(&:strip)
    when /^\[/
      ['', part.delete('[]').strip]
    else
      [part]
    end
  }

  f = ->(a, *as, &b) {
    if as.empty?
      a.each { |x|
        b.([x])
      }
    else
      a.each { |x|
        f.(*as) { |xs|
          b.([x, *xs])
        }
      }
    end
  }

  f.(*arrays) { |words|
    yield words.reject(&:empty?).join(' ')
  }
end

DOCSET = 'BigQuery Standard SQL.docset'
DOCSET_ARCHIVE = File.basename(DOCSET, '.docset') + '.tgz'
DOCS_ROOT = File.join(DOCSET, 'Contents/Resources/Documents')
DOCS_URI = URI('https://cloud.google.com/bigquery/docs/reference/standard-sql/')
HOST_URI = DOCS_URI + '/'
DL_DIR = Pathname(DOCS_URI.host)
ICON_SITE_URI = URI('https://cloudplatform-jp.googleblog.com/2015/04/google-bigquery.html')
ICON_FILE = Pathname('icon.png')
COMMON_CSS = Pathname('common.css')
FETCH_LOG = 'wget.log'

def extract_version
  version = ''

  Dir.chdir(DOCS_ROOT) {
    Dir.glob("#{HOST_URI.route_to(DOCS_URI)}*.html") { |path|
      doc = Nokogiri::HTML(File.read(path), path)
      version = [version, Time.parse(doc.at('//p[@itemprop="datePublished"]/@content').value).strftime('%Y.%m.%d')].max
    }
  }

  version
end

desc 'Fetch the BigQuery Standard SQL document files.'
task :fetch => [DL_DIR, ICON_FILE]

file DL_DIR do |t|
  puts 'Downloading %s' % DOCS_URI
  sh 'wget', '-nv', '--append-output', FETCH_LOG, '-r', '--no-parent', '-nc', '-p',
     '--reject-regex=\?hl=', '--reject-regex=/standard-sql/[^./]+$', DOCS_URI.to_s

  # Google responds with gzip'd asset files despite wget's sending
  # `Accept-Encoding: identity`, and wget has no capability in
  # decoding gzip contents.
  Dir.glob("#{t.name}/**/*") { |path|
    next unless File.file?(path)
    begin
      data = Zlib::GzipReader.open(path, &:read)
    rescue Zlib::GzipFile::Error
    else
      puts "Uncompressing #{path}"
      File.write(path, data)
    end
  }
end

file ICON_FILE do |t|
  Tempfile.create(['icon', '.png']) { |temp|
    agent = Mechanize.new
    page = agent.get(ICON_SITE_URI)
    image = page.image_with(xpath: '//img[@title="Google BigQuery"]')
    image.fetch.save!(temp.path)
    sh 'sips', '-z', '64', '64', temp.path, '--out', t.name
  }
end

desc 'Build a docset in the current directory.'
task :build => :fetch do |t|
  target = DOCSET

  rm_rf target

  mkdir_p DOCS_ROOT

  cp 'Info.plist', File.join(target, 'Contents')
  cp ICON_FILE, target

  cp_r DL_DIR.to_s + '/.', DOCS_ROOT

  # Index
  db = SQLite3::Database.new(File.join(target, 'Contents/Resources/docSet.dsidx'))

  db.execute(<<-SQL)
    CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
    CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);
  SQL

  insert = db.prepare(<<-SQL)
    INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?, ?, ?);
  SQL

  puts 'Indexing documents'

  cp COMMON_CSS, File.join(DOCS_ROOT, HOST_URI.route_to(DOCS_URI).to_s)

  Dir.chdir(DOCS_ROOT) {
    Dir.glob("#{HOST_URI.route_to(DOCS_URI)}*.html") { |path|
      uri = HOST_URI + path
      doc = Nokogiri::HTML(File.read(path), path)

      doc.xpath('//meta[@property or @name] | //script | //link[not(@rel="stylesheet")]').each(&:remove)

      [
        ['a', 'href'],
        ['img', 'src'],
        ['link', 'href'],
        ['script', 'src'],
      ].each { |tag, attr|
        doc.css('%s[%s^="%s"]' % [tag, attr, HOST_URI.to_s]).each { |e|
          abs = e[attr]
          filepath = HOST_URI.route_to(abs).to_s
          case filepath
          when 'bigquery/sql-reference/'
            e[attr] = uri.route_to(DOCS_URI + 'index.html').to_s
          else
            if File.file?(filepath)
              e[attr] = uri.route_to(abs).to_s
            elsif File.directory?(filepath) &&
                  File.file?(File.join(filepath.chomp('/'), 'index.html'))
              e[attr] = uri.route_to(File.join(abs.chomp('/'), 'index.html')).to_s
            end
          end
        }
      }

      doc.xpath('//link[contains(@href, "//")]').each(&:remove)

      if article = doc.at('article.devsite-article')
        doc.at('body').inner_html = article.inner_html
      end

      link = Nokogiri::XML::Node.new('link', doc)
      link['rel'] = 'stylesheet'
      link['href'] = COMMON_CSS.basename.to_s
      doc.at('head') << link

      if h1 = doc.at('h1')
        type = h1['id'] = 'h1'
        insert.execute(h1.xpath('normalize-space(.)'), 'Section', "#{path}\##{type}")
      end

      case basename = File.basename(path)
      when 'enabling-standard-sql.html'
        doc.css('h2[id]').each { |h|
          id = h['id']
          title = h.xpath('normalize-space(.)')
          insert.execute(title, 'Section', "#{path}\##{id}")

          case id
          when 'sql-prefix'
            h.xpath('./following-sibling::table[1]/tbody/tr/td[1]').each { |td|
              directive = td.xpath('normalize-space(.)')
              insert.execute(directive, 'Directive', "#{path}\##{id}")
            }
          end
        }
      when 'data-types.html'
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          id = h['id']
          case title = h.xpath('normalize-space(.)')
          when 'Format', 'Canonical format', 'Examples'
            next
          else
            insert.execute(title, 'Section', "#{path}\##{id}")
          end
        }
        doc.css('h2[id$="-type"]').each { |h|
          id = h['id']
          h.xpath('./following-sibling::table[1]/tbody/tr/td[1]').each { |td|
            type = td.xpath('normalize-space(.)')
            insert.execute(type, 'Type', "#{path}\##{id}")
          }
        }
      when 'functions-and-operators.html'
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          id = h['id']
          case title = h.xpath('normalize-space(.)')
          when /\A(?<func>[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)( and \g<func>)*(?: operators?)?\z/
            type = h.name == 'h6' ? 'Query' : 'Function'
            title.scan(/[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*/) { |name|
              insert.execute(name, type, "#{path}\##{id}")
            }
            next
          when /\A(Arithmetic|Bitwise|Logical|Comparison) operators\z/
            h.xpath('./following-sibling::table[1]/tbody/tr/td[2]/text()').each { |text|
              syntax = text.xpath('normalize-space(.)')
              next unless /\bX\b/ === syntax
                op, rop, = syntax.gsub!(/\b[XYZ]\b/, '').split
              case op
              when '[NOT]'
                insert.execute(rop, 'Operator', "#{path}\##{id}")
                insert.execute("NOT #{rop}", 'Operator', "#{path}\##{id}")
              else
                insert.execute(op, 'Operator', "#{path}\##{id}")
              end
            }
          when 'Element access operators'
            h.xpath('./following-sibling::table[1]/tbody/tr/td[1]/text()').each { |text|
              syntax = text.xpath('normalize-space(.)').delete(' ')  # "[ ]" -> "[]"
              insert.execute(syntax, 'Operator', "#{path}\##{id}")
            }
          when / operators\z/
            raise "#{path}: Unknown section: #{title}"
          when 'Casting'
            insert.execute('CAST', 'Function', "#{path}\##{id}")
          when 'Safe casting'
            insert.execute('SAFE_CAST', 'Function', "#{path}\##{id}")
          end
          insert.execute(title, 'Section', "#{path}\##{id}")
        }
      else
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          id = h['id']
          case title = h.xpath('normalize-space(.)')
          when 'SQL Syntax'
            if basename == 'query-syntax.html'
              insert.execute('SELECT', 'Statement', "#{path}\##{id}")
            end
          when 'External UDF structure'
            statement = h.xpath('string(./following-sibling::ul[1]/li[1]/strong)')
            statement
          when 'Syntax'
            next
          when /\A(?<ws>(?<w>[A-Z]+)(?: \g<w>)*) (?<t>statement|keyword|clause)(?: and \g<ws> \k<t>)?\z/
            type = $~[:t] == 'statement' ? 'Statement' : 'Query'
            title.scan(/(?<ws>\G(?<w>[A-Z]+)(?: \g<w>)*)/) { |ws,|
              insert.execute(ws, type, "#{path}\##{id}")
            }
          when /\A(?:(?<w>[A-Z]+) )*(?<ow>\[\g<w>\] )?\g<w>\z/
            optional = !!$~[:ow]
            type =
              case title
              when /\ASELECT\b/
                'Statement'
              when /\bJOIN\z/, 'UNION'
                'Query'
              when 'UNNEST'
                'Function'
              else
                raise "#{path}: Unknown directive: #{title}"
              end
            expand(title) { |query|
              insert.execute(query, type, "#{path}\##{id}")
            }
          end
          insert.execute(title, 'Section', "#{path}\##{id}")
        }
      end

      File.write(path, doc.to_s)
    }
  }

  get_count = ->(**criteria) do
    db.get_first_value(<<-SQL, criteria.values)
      SELECT COUNT(*) from searchIndex where #{
        criteria.each_key.map { |column| "#{column} = ?" }.join(' and ')
      }
    SQL
  end

  assert_exists = ->(**criteria) do
    if get_count.(**criteria).zero?
      raise "#{criteria.inspect} not found in index!"
    end
  end

  puts 'Performing sanity check'

  {
    'Directive' => %w[#legacySQL #standardSQL],
    'Statement' => ['SELECT', 'INSERT', 'INSERT SELECT', 'UPDATE', 'DELETE'],
    'Query' => ['JOIN', 'INNER JOIN', 'GROUP BY', 'LIMIT'],
    'Function' => ['CAST', 'SAFE_CAST', 'UNNEST'],
    'Operator' => ['+', '~', '^', '<=', '!=', '<>', '.', '[]',
                   'BETWEEN', 'NOT LIKE', 'AND', 'OR', 'NOT'],
  }.each { |type, names|
    names.each { |name|
      assert_exists.(name: name, type: type)
    }
  }

  puts "Finished creating #{target} version #{extract_version()}"
end

task :prepare, [:workdir] do |t, args|
  version = extract_version
  workdir = Pathname(args[:workdir] || '../Dash-User-Contributions') / 'docsets/BigQuery_Standard_SQL'

  docset_json = workdir / 'docset.json'
  archive = workdir / DOCSET_ARCHIVE
  versioned_archive = workdir / 'versions' / version / DOCSET_ARCHIVE

  sh 'tar', '-zcf', DOCSET_ARCHIVE, '--exclude=.DS_Store', DOCSET and
    mv DOCSET_ARCHIVE, archive and
    mkdir_p versioned_archive.dirname and
    cp archive, versioned_archive

  puts "Updating #{docset_json}"
  File.open(docset_json, 'r+') { |f|
    json = JSON.parse(f.read)
    json['version'] = version
    specific_version = {
      'version' => version,
      'archive' => versioned_archive.relative_path_from(workdir).to_s
    }
    json['specific_versions'] = [specific_version] | json['specific_versions']
    f.rewind
    f.puts JSON.pretty_generate(json, indent: "    ")
    f.truncate(f.tell)
  }

  Dir.chdir(workdir.to_s) {
    sh 'git', 'add', *[archive, versioned_archive, docset_json].map { |path|
      path.relative_path_from(workdir).to_s
    }
  }
end

desc 'Delete all fetched files and generated files'
task :clean do
  rm_rf [DL_DIR, ICON_FILE, DOCSET, DOCSET_ARCHIVE, FETCH_LOG]
end

task :default => :build
