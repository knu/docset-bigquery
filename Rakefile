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

DOCSET_NAME = 'BigQuery Standard SQL'
DOCSET = "#{DOCSET_NAME.tr(' ', '_')}.docset"
DOCSET_ARCHIVE = File.basename(DOCSET, '.docset') + '.tgz'
DOCS_ROOT = File.join(DOCSET, 'Contents/Resources/Documents')
DOCS_URI = URI('https://cloud.google.com/bigquery/docs/reference/standard-sql/')
HOST_URI = DOCS_URI + '/'
DL_DIR = Pathname(DOCS_URI.host)
ICON_SITE_URI = URI('https://cloudplatform-jp.googleblog.com/2015/04/google-bigquery.html')
ICON_FILE = Pathname('icon.png')
COMMON_CSS = Pathname('common.css')
COMMON_CSS_URL = DOCS_URI + COMMON_CSS.basename.to_s
FETCH_LOG = 'wget.log'
DUC_REPO = 'git@github.com:knu/Dash-User-Contributions.git'
DUC_REPO_UPSTREAM = 'https://github.com/Kapeli/Dash-User-Contributions.git'
DUC_WORKDIR = File.basename(DUC_REPO, '.git')
DUC_BRANCH = 'bigquery_standard_sql'

URI_ATTRS = [
  ['a', 'href'],
  ['img', 'src'],
  ['link', 'href'],
  ['script', 'src'],
]
FILE_SUFFIXES = [
  '',
  '/index.html',
  '.html'
]

def extract_version
  version = ''

  cd DOCS_ROOT do
    Dir.glob("#{HOST_URI.route_to(DOCS_URI)}*.html") { |path|
      doc = Nokogiri::HTML(File.read(path), path)
      version = [version, Time.parse(doc.at('//p[@itemprop="datePublished"]/@content').value).strftime('%Y.%m.%d')].max
    }
  end

  version
end

desc "Fetch the #{DOCSET_NAME} document files."
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
    sh 'type', 'sips' do |ok, _res|
      if ok
        sh 'sips', '-z', '64', '64', temp.path, '--out', t.name
      else
        sh 'convert', '-resample', '64x64', temp.path, t.name
      end
    end
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

  index_item = ->(path, node, type, name) {
    a = Nokogiri::XML::Node.new('a', node.document)
    a['name'] = id = '//apple_ref/cpp/%s/%s' % [type, name].map { |s|
      URI.encode_www_form_component(s).gsub('+', '%20')
    }
    a['class'] = 'dashAnchor'
    node.prepend_child(a)
    url = "#{path}\##{id}"
    insert.execute(name, type, url)
  }

  puts 'Indexing documents'

  cp COMMON_CSS, File.join(DOCS_ROOT, HOST_URI.route_to(DOCS_URI).to_s)

  cd DOCS_ROOT do
    Dir.glob("#{HOST_URI.route_to(DOCS_URI)}*.html") { |path|
      uri = HOST_URI + path
      doc = Nokogiri::HTML(File.read(path), path)

      doc.xpath('//meta[not(@charset or @name = "viewport")] | //script | //link[not(@rel="stylesheet")]').each(&:remove)

      URI_ATTRS.each { |tag, attr|
        doc.css("#{tag}[#{attr}]").each { |e|
          abs = uri + e[attr]
          rel = HOST_URI.route_to(abs)
          next if rel.host || %r{\A\.\./} === rel.path
          case localpath = rel.path.chomp('/')
          when 'bigquery/sql-reference'
            abs.path = (DOCS_URI + 'index.html').path
            e[attr] = uri.route_to(abs)
          else
            FILE_SUFFIXES.each { |suffix|
              if File.file?(localpath + suffix)
                abs.path = abs.path.chomp('/') + suffix
                e[attr] = uri.route_to(abs)
                break
              end
            }
          end
        }
      }

      doc.xpath('//link[contains(@href, "//")]').each(&:remove)

      if article = doc.at('article.devsite-article')
        doc.at('body').inner_html = article.inner_html
      end

      link = Nokogiri::XML::Node.new('link', doc)
      link['rel'] = 'stylesheet'
      link['href'] = uri.route_to(COMMON_CSS_URL)
      doc.at('head') << link

      if h1 = doc.at('h1')
        index_item.(path, h1, 'Section', h1.xpath('normalize-space(.)'))
      end

      case basename = File.basename(path)
      when 'enabling-standard-sql.html'
        doc.css('h2[id]').each { |h|
          id = h['id']
          title = h.xpath('normalize-space(.)')
          index_item.(path, h, 'Section', title)

          case id
          when 'sql-prefix'
            h.xpath('./following-sibling::table[1]/tbody/tr/td[1]').each { |td|
              directive = td.xpath('normalize-space(.)')
              index_item.(path, td, 'Directive', directive)
            }
          end
        }
      when 'data-types.html'
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          case title = h.xpath('normalize-space(.)')
          when 'Format', 'Canonical format', 'Examples'
            next
          else
            index_item.(path, h, 'Section', title)
          end
        }
        doc.css('h2[id$="-type"]').each { |h|
          h.xpath('./following-sibling::table[1]/tbody/tr/td[1]').each { |td|
            type = td.xpath('normalize-space(.)')
            index_item.(path, td, 'Type', type)
          }
        }
      when 'functions-and-operators.html'
        doc.xpath('//table/thead/tr[1]/th[position() = 1 and (text() = "Syntax" or text() = "Function")]').each { |th|
          th.xpath('./ancestor::table[1]/tbody/tr/td[1]').each { |td|
            case text = td.xpath('normalize-space(.)')
            when /\A([A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)\(/
              index_item.(path, td, 'Function', $1)
            when /\A(CASE(?: WHEN)?) /
              index_item.(path, td, 'Function', $1)
            else
              raise "#{path}: Unknown function: #{text}"
            end
          }
        }

        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          case title = h.xpath('normalize-space(.)')
          when /\A(?<func>[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)( and \g<func>)*(?: operators?)?\z/
            type = h.name == 'h6' ? 'Query' : 'Function'
            title.scan(/[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*/) { |name|
              index_item.(path, h, type, name)
            }
            next
          when /\A(Arithmetic|Bitwise|Logical|Comparison) operators\z/
            h.xpath('./following-sibling::table[1]/tbody/tr/td[2]/text()').each { |text|
              syntax = text.xpath('normalize-space(.)')
              next unless /\bX\b/ === syntax
              op, rop, = syntax.gsub!(/\b[XYZ]\b/, '').split
              case op
              when '[NOT]'
                index_item.(path, h, 'Operator', rop)
                index_item.(path, h, 'Operator', "NOT #{rop}")
              else
                index_item.(path, h, 'Operator', op)
              end
            }
          when 'Element access operators'
            h.xpath('./following-sibling::table[1]/tbody/tr/td[1]/text()').each { |text|
              syntax = text.xpath('normalize-space(.)').delete(' ')  # "[ ]" -> "[]"
              index_item.(path, h, 'Operator', syntax)
            }
          when / operators\z/
            raise "#{path}: Unknown section: #{title}"
          when 'Casting'
            index_item.(path, h, 'Function', 'CAST')
          when 'Safe casting'
            index_item.(path, h, 'Function', 'SAFE_CAST')
          end
          index_item.(path, h, 'Section', title)
        }
      else
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          case title = h.xpath('normalize-space(.)')
          when 'SQL Syntax'
            if basename == 'query-syntax.html'
              index_item.(path, h, 'Statement', 'SELECT')
            end
          when 'External UDF structure'
            statement_node, *query_nodes = h.xpath('./following-sibling::ul[1]/li/strong')
            expand(statement_node.text) { |statement|
              index_item.(path, statement_node, 'Statement', statement)
            }
            query_nodes.each { |query_node|
              index_item.(path, query_node, 'Query', query_node.text)
            }
          when 'Syntax'
            next
          when /\A(?<ws>(?<w>[A-Z]+)(?: \g<w>)*) (?<t>statement|keyword|clause)(?: and \g<ws> \k<t>)?\z/
            type = $~[:t] == 'statement' ? 'Statement' : 'Query'
            title.scan(/(?<ws>\G(?<w>[A-Z]+)(?: \g<w>)*)/) { |ws,|
              index_item.(path, h, type, ws)
            }
          when /\A(?:(?<w>[A-Z]+) )*(?<ow>\[\g<w>\] )?\g<w>\z/
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
              index_item.(path, h, type, query)
            }
          end
          index_item.(path, h, 'Section', title)
        }
      end

      File.write(path, doc.to_s)
    }
  end

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
    'Function' => ['CAST', 'SAFE_CAST', 'UNNEST',
                   'CASE', 'CASE WHEN', 'COALESCE', 'NULLIF',
                   'DENSE_RANK', 'CUME_DIST',
                   'GREATEST', 'LOG10',
                   'COS', 'ASINH',
                   'FLOOR'],
    'Operator' => ['+', '~', '^', '<=', '!=', '<>', '.', '[]',
                   'BETWEEN', 'NOT LIKE', 'AND', 'OR', 'NOT'],
  }.each { |type, names|
    names.each { |name|
      assert_exists.(name: name, type: type)
    }
  }

  puts "Finished creating #{target} version #{extract_version()}"
end

file DUC_WORKDIR do |t|
  sh 'git', 'clone', DUC_REPO, t.name
  cd t.name do
    sh 'git', 'remote', 'add', 'upstream', DUC_REPO_UPSTREAM
    sh 'git', 'remote', 'update', 'upstream'
  end
end

desc 'Push the generated docset if there is an update'
task :push => DUC_WORKDIR do
  version = extract_version
  workdir = Pathname(DUC_WORKDIR) / 'docsets' / File.basename(DOCSET, '.docset')

  docset_json = workdir / 'docset.json'
  archive = workdir / DOCSET_ARCHIVE
  versioned_archive = workdir / 'versions' / version / DOCSET_ARCHIVE

  puts "Resetting the working directory"
  cd workdir.to_s do
    sh 'git', 'remote', 'update'
    sh 'git', 'checkout', DUC_BRANCH
    sh 'git', 'reset', '--hard', 'upstream/master'
  end

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

  cd workdir.to_s do
    sh 'git', 'diff', '--exit-code', docset_json.relative_path_from(workdir).to_s do |ok, _res|
      if ok
        puts "Nothing to commit."
      else
        sh 'git', 'add', *[archive, versioned_archive, docset_json].map { |path|
          path.relative_path_from(workdir).to_s
        }
        sh 'git', 'commit', '-m', "Update #{DOCSET_NAME} docset to #{version}"
        sh 'git', 'push', '-f', 'origin', DUC_BRANCH
      end
    end
  end
end

desc 'Delete all fetched files and generated files'
task :clean do
  rm_rf [DL_DIR, ICON_FILE, DOCSET, DOCSET_ARCHIVE, FETCH_LOG]
end

task :default => :build
