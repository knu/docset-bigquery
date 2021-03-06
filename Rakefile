# frozen_string_literal: true
require 'bundler/setup'
Bundler.require

require 'json'
require 'pathname'
require 'tempfile'
require 'time'
require 'uri'
require 'zlib'
require 'rubygems/version'

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
ROOT_RELPATH = 'Contents/Resources/Documents'
INDEX_RELPATH = 'Contents/Resources/docSet.dsidx'
DOCS_ROOT = File.join(DOCSET, ROOT_RELPATH)
DOCS_INDEX = File.join(DOCSET, INDEX_RELPATH)
DOCS_URI = URI('https://cloud.google.com/bigquery/docs/reference/standard-sql/')
HOST_URI = DOCS_URI + '/'
DL_DIR = Pathname(DOCS_URI.host)
ICON_SITE_URI = URI('https://cloudplatform-jp.googleblog.com/2015/04/google-bigquery.html')
ICON_FILE = Pathname('icon.png')
COMMON_CSS = Pathname('common.css')
COMMON_CSS_URL = DOCS_URI + COMMON_CSS.basename.to_s
FETCH_LOG = 'wget.log'
DUC_OWNER = 'knu'
DUC_REPO = "git@github.com:#{DUC_OWNER}/Dash-User-Contributions.git"
DUC_OWNER_UPSTREAM = 'Kapeli'
DUC_REPO_UPSTREAM = "https://github.com/#{DUC_OWNER_UPSTREAM}/Dash-User-Contributions.git"
DUC_WORKDIR = File.basename(DUC_REPO, '.git')
DUC_BRANCH = 'bigquery'

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
      if date_published = doc.at('//*[@itemprop="datePublished"]/@content')&.value ||
          doc.at('//devsite-content-footer//p[starts-with(., "Last updated ")]/text()')&.then { |t|
            t.to_s[/\b\d{4}-\d{2}-\d{2}\b/]
          }
        version = [version, Time.parse(date_published).strftime('%Y.%m.%d')].max
      end
    }
  end

  version
end

def previous_version
  current_version = Gem::Version.new(extract_version)
  Pathname.glob("versions/*/#{DOCSET}").map { |path|
    Gem::Version.new(path.parent.basename.to_s)
  }.select { |version|
    version < current_version
  }.max&.to_s
end

def previous_docset
  version = previous_version or raise 'No previous version found'

  "versions/#{version}/#{DOCSET}"
end

desc "Fetch the #{DOCSET_NAME} document files."
task :fetch => %i[fetch:icon fetch:docs]

namespace :fetch do
  task :docs do
    puts 'Downloading %s' % DOCS_URI
    sh 'wget', '-nv', '--append-output', FETCH_LOG, '-r', '--no-parent', '-N', '-p',
       '--reject-regex=\?hl=|://cloud\.google\.com/images/(artwork|backgrounds|home|icons|logos)/|\.md$',
       DOCS_URI.to_s

    # Google responds with gzip'd asset files despite wget's sending
    # `Accept-Encoding: identity`, and wget has no capability in
    # decoding gzip contents.
    Dir.glob("#{DL_DIR}/**/*") { |path|
      next unless File.file?(path)
      begin
        data = Zlib::GzipReader.open(path, &:read)
      rescue Zlib::GzipFile::Error
      else
        puts "Uncompressing #{path}"
        File.write(path, data)
      end

      if !path.end_with?('.html') &&
          File.open(path) { |f| f.read(255).match?(/<!DOCTYPE html>/i) }
        path_with_suffix = path + '.html'
        if File.file?(path_with_suffix)
          rm path
        else
          mv path, path_with_suffix
        end
      end
    }
  end

  task :icon do 
    Tempfile.create(['icon', '.png']) { |temp|
      agent = Mechanize.new
      page = agent.get(ICON_SITE_URI)
      image = page.image_with(xpath: '//img[@title="Google BigQuery"]')
      image.fetch.save!(temp.path)
      sh 'type', 'sips' do |ok, _res|
        if ok
          sh 'sips', '-z', '64', '64', temp.path, '--out', ICON_FILE.to_s
        else
          sh 'convert', '-resample', '64x64', temp.path, ICON_FILE.to_s
        end
      end
    }
  end
end

file DL_DIR do
  Rake::Task[:'fetch:docs'].invoke
end

file ICON_FILE do
  Rake::Task[:'fetch:icon'].invoke
end

desc 'Build a docset in the current directory.'
task :build => [DL_DIR, ICON_FILE] do |t|
  rm_rf DOCSET

  mkdir_p DOCS_ROOT

  cp 'Info.plist', File.join(DOCSET, 'Contents')
  cp ICON_FILE, DOCSET

  cp_r DL_DIR.to_s + '/.', DOCS_ROOT

  version = extract_version or raise "Version unknown"

  puts "Generating docset for #{DOCSET_NAME} #{version}"

  # Index
  db = SQLite3::Database.new(DOCS_INDEX)

  db.execute(<<-SQL)
    CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
    CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);
  SQL

  insert = db.prepare(<<-SQL)
    INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?, ?, ?);
  SQL

  index_item = ->(path, node, type, name) {
    id = '//apple_ref/cpp/%s/%s' % [type, name].map { |s|
      URI.encode_www_form_component(s).gsub('+', '%20')
    }
    node.prepend_child(
      Nokogiri::XML::Node.new('a', node.document).tap { |a|
        a['name'] = id
        a['class'] = 'dashAnchor'
      }
    )
    url = "#{path}\##{id}"
    insert.execute(name, type, url)
  }

  puts 'Indexing documents'

  cp COMMON_CSS, File.join(DOCS_ROOT, HOST_URI.route_to(DOCS_URI).to_s)

  cd DOCS_ROOT do
    Dir.glob("#{HOST_URI.route_to(DOCS_URI)}*.html") { |path|
      uri = HOST_URI + path
      doc = Nokogiri::HTML(File.read(path), path)

      if /\b(\d{4}-\d{2}-\d{2})\b/ === doc.at('//devsite-content-footer//p[starts-with(., "Last updated ")]/text()')&.to_s
        last_updated = $1
      end

      doc.xpath('//meta[not(@charset or @name = "viewport")] | //script | //link[not(@rel="stylesheet")]').each(&:remove)

      URI_ATTRS.each { |tag, attr|
        doc.css("#{tag}[#{attr}]").each { |e|
          abs = uri + e[attr].sub(/\.md\z/, '')
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

      doc.at('head') << Nokogiri::XML::Node.new('link', doc).tap { |link|
        link['rel'] = 'stylesheet'
        link['href'] = uri.route_to(COMMON_CSS_URL)
      }

      if last_updated
        doc.at('body') << Nokogiri::XML::Node.new('span', doc).tap { |span|
          span['itemprop'] = 'datePublished'
          span['content'] = "#{last_updated}T00:00:00Z"
        }
      end

      if h1 = doc.at('h1')
        index_item.(path, h1, 'Section', h1.xpath('normalize-space(.)'))
      end

      case basename = File.basename(path)
      when 'index.html'
        doc.css('h2[id]').each { |h|
          id = h['id']
          title = h.xpath('normalize-space(.)')
          index_item.(path, h, 'Section', title)

          case id
          when 'sql-prefix'
            h.xpath('(./following-sibling::*/descendant-or-self::table)[1]/tbody/tr/td[1]').each { |td|
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
          h.xpath('(./following-sibling::*/descendant-or-self::table)[1]/tbody/tr/td[1]').each { |td|
            type = td.xpath('normalize-space(.)')
            index_item.(path, td, 'Type', type)
          }
        }
        doc.xpath('//table/thead/tr[1]/th[position() = 1 and text() = "Name"]').each { |th|
          th.xpath('./ancestor::table[1]/tbody/tr/td[1]').each { |td|
            case text = td.xpath('normalize-space(.)')
            when /\A([A-Z][A-Z0-9]*)(?: \(Preview\))?\z/
              index_item.(path, td, 'Type', $1)
            else
              raise "#{path}: Unknown type: #{text}"
            end
          }
        }
      when 'functions-and-operators.html'
        # This page is now a one-page function reference that wraps up
        # the following pages.
      when /_functions\.html\z/, 'conversion_rules.html', 'conditional_expressions.html', 'operators.html'
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
          when /\A(?<func>[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)( (?:and|or) \g<func>)*(?: (?:operators?|expr))?\z/
            # 'or' is for 'JSON_EXTRACT or JSON_EXTRACT_SCALAR'
            type = h.name == 'h6' ? 'Query' : 'Function'
            title.scan(/[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*/) { |name|
              index_item.(path, h, type, name)
            }
            next
          when 'Logical operators'
            h.xpath('(./following-sibling::p)[1]/code').each { |code|
              case text = code.text
              when 'TRUE', 'FALSE', 'NULL'
                # skip
              when 'AND', 'OR', 'NOT'
                index_item.(path, h, 'Operator', text)
              else
                raise "#{path}: Unknown loginal operator: #{text}"
              end
            }
          when /\A(Arithmetic|Bitwise|Logical|Comparison) operators\z/
            h.xpath('(./following-sibling::*/descendant-or-self::table)[1]/tbody/tr/td[2]/text()').each { |text|
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
            h.xpath('(./following-sibling::*/descendant-or-self::table)[1]/tbody/tr/td[1]/text()').each { |text|
              syntax = text.xpath('normalize-space(.)').delete(' ')  # "[ ]" -> "[]"
              index_item.(path, h, 'Operator', syntax)
            }
          when 'Date arithmetics operators'
            # Nothing to link
          when 'Concatenation operator'
            index_item.(path, h, 'Operator', '||')
          when / operators?\z/
            raise "#{path}: Unknown section: #{title}"
          when 'Casting'
            index_item.(path, h, 'Function', 'CAST')
          when 'Safe casting'
            index_item.(path, h, 'Function', 'SAFE_CAST')
          end
          index_item.(path, h, 'Section', title)
        }
      when 'scripting.html'
        doc.css('h2[id], h3[id]').each { |h|
          case title = h.xpath('normalize-space(.)')
          when /\A[A-Z]{2,}\b/
            index_item.(path, h, 'Statement', title)
          else
            index_item.(path, h, 'Section', title)
          end
        }
      when 'aead-encryption-concepts.html'
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          title = h.xpath('normalize-space(.)')
          index_item.(path, h, 'Section', title)
        }
      when 'subqueries.html'
        doc.css('h2[id], h3[id]').each { |h|
          case title = h.xpath('normalize-space(.)')
          when /\A([A-Z]{2,}) subqueries\b/
            index_item.(path, h, 'Query', $1)
          else
            index_item.(path, h, 'Section', title)
          end
        }
      else
        doc.css('h2[id], h3[id], h4[id], h5[id], h6[id]').each { |h|
          next if h.at_xpath('./ancestor::*[contains(@class, "ds-selector-tabs")]')

          case title = h.xpath('normalize-space(.)')
          when 'SQL syntax'
            if basename == 'query-syntax.html'
              index_item.(path, h, 'Statement', 'SELECT')
            end
          when 'UDF syntax'
            statement_node, *query_nodes = h.xpath('./following-sibling::ul[1]/li/p/strong')
            expand(statement_node.text) { |statement|
              index_item.(path, statement_node, 'Statement', statement)
            }
            query_nodes.each { |query_node|
              index_item.(path, query_node, 'Query', query_node.text)
            }
          when 'Syntax'
            next
          when 'Defining the window frame clause'
            code, = h.xpath('./following-sibling::pre[1]/code')
            code.text.scan(/[A-Z]+(?: [A-Z]+)*/) { |query|
              index_item.(path, code, 'Query', query) unless query == 'AND'
            }
          when /\A(?<ws>(?<w>[A-Z]+)(?: \g<w>)*) (?<t>statement|keyword|[cC]lause)(?: and \g<ws> \k<t>)?\z/
            type = $~[:t] == 'statement' ? 'Statement' : 'Query'
            title.scan(/(?<ws>\G(?<w>[A-Z]+)(?: \g<w>)*)/) { |ws,|
              index_item.(path, h, type, ws)
            }
          when %r{
            \A
            (?# at least one capital word block )
            (?= .* (?<ws> (?<w>[A-Z]+)(?: \g<w>)*) )
            (?# at least one type name )
            (?= .* (?<type> (?<t>[sS]tatement)s? | (?<t>[kK]eyword)s? | (?<t>[cC]lause)s? ) )
            (?<token> \g<ws> | and | \g<type> ) (?: [ ] \g<token> )*
            \z
          }x
            type = $~[:t] == 'statement' ? 'Statement' : 'Query'
            title.scan(/(?<ws>(?<w>[A-Z]+)(?: \g<w>)*)/) { |ws,|
              index_item.(path, h, type, ws)
            }
          when /\A(?:(?<w>(?:[A-Z]+_)*[A-Z]+) )*(?<ow>\[\g<w>\] )?\g<w>\z/
            type =
              case title
              when /\A(?:(?:SELECT|CREATE|ASSERT)\b|(?:)\z)/
                'Statement'
              when /\bJOIN\z/, 'UNION', 'INTERSECT', 'EXCEPT', 'FOR SYSTEM_TIME AS OF',
                  'ROWS', 'RANGE'
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

  insert.close

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
    'Statement' => ['SELECT', 'INSERT', 'INSERT SELECT', 'UPDATE', 'DELETE',
                    'CREATE PROCEDURE', 'DECLARE', 'IF', 'WHILE', 'ASSERT',
                    'CREATE EXTERNAL TABLE', 'EXPORT DATA'],
    'Query' => ['JOIN', 'INNER JOIN',
                'UNION', 'INTERSECT', 'EXCEPT',
                'FOR SYSTEM_TIME AS OF',
                'GROUP BY', 'LIMIT', 'OFFSET',
                'ARRAY', 'IN', 'EXISTS',
                'ROWS', 'RANGE', 'BETWEEN',
                'UNBOUNDED FOLLOWING', 'FOLLOWING',
                'CURRENT ROW'],
    'Function' => ['CAST', 'SAFE_CAST', 'UNNEST',
                   'AEAD.ENCRYPT', 'KEYS.NEW_KEYSET',
                   'ARRAY_AGG', 'COUNTIF', 'LOGICAL_AND', 'MAX',
                   'APPROX_COUNT_DISTINCT',
                   'ARRAY', 'ARRAY_REVERSE', 'SAFE_OFFSET',
                   'BIT_COUNT',
                   'DATE_DIFF', 'UNIX_DATE',
                   'DATETIME_DIFF',
                   'ERROR',
                   'ST_DWITHIN',
                   'SHA256',
                   'HLL_COUNT.MERGE',
                   'JSON_EXTRACT',
                   'COS', 'GREATEST', 'TRUNC',
                   'NTH_VALUE', 'LAG',
                   'NET.IP_NET_MASK',
                   'DENSE_RANK', 'CUME_DIST',
                   'SESSION_USER',
                   'CORR', 'STDDEV',
                   'CHARACTER_LENGTH', 'TO_BASE64',
                   'TIME_DIFF',
                   'TIMESTAMP_DIFF',
                   'GENERATE_UUID',
                   'CASE', 'COALESCE', 'NULLIF'],
    'Type' => ['INT64', 'FLOAT64', 'NUMERIC', 'BOOL', 'STRING', 'BYTES',
               'DATE', 'DATETIME', 'TIME', 'TIMESTAMP',
               'ARRAY', 'STRUCT', 'BIGNUMERIC'],
    'Operator' => ['+', '~', '^', '<=', '!=', '<>', '.', '[]', '||',
                   'BETWEEN', 'NOT LIKE', 'AND', 'OR', 'NOT'],
    'Section' => ['GCM', 'Loops', 'UDF syntax']
  }.each { |type, names|
    names.each { |name|
      assert_exists.(name: name, type: type)
    }
  }

  db.close

  mkdir_p "versions/#{version}/#{DOCSET}"
  sh 'rsync', '-a', '--exclude=.DS_Store', '--delete', "#{DOCSET}/", "versions/#{version}/#{DOCSET}/"

  puts "Finished creating #{DOCSET} #{version}"

  Rake::Task[:diff].invoke
end

task :diff do
  system "rake diff:index diff:docs | #{ENV['PAGER'] || 'more'}"
end

namespace :diff do
  desc 'Show the differences in the index from an installed version.'
  task :index do
    old_index = File.join(previous_docset, INDEX_RELPATH)

    begin
      sql = "SELECT name, type, path FROM searchIndex ORDER BY name, type, path"

      odb = SQLite3::Database.new(old_index)
      ndb = SQLite3::Database.new(DOCS_INDEX)

      Tempfile.create(['old', '.txt']) { |otxt|
        odb.execute(sql) { |row|
          otxt.puts row.join("\t")
        }
        odb.close
        otxt.close

        Tempfile.create(['new', '.txt']) { |ntxt|
          ndb.execute(sql) { |row|
            ntxt.puts row.join("\t")
          }
          ndb.close
          ntxt.close

          sh 'diff', '-U3', otxt.path, ntxt.path do
            # ignore status
          end
        }
      }
    ensure
      odb&.close
      ndb&.close
    end
  end

  desc 'Show the differences in the docs from an installed version.'
  task :docs do
    old_root = File.join(previous_docset, ROOT_RELPATH)

    sh 'diff', '-rNU3',
      '-x', '*.js',
      '-x', '*.css',
      '-x', '*.svg',
      old_root, DOCS_ROOT do
      # ignore status
    end
  end
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
    sh 'git', 'rev-parse', '--verify', '--quiet', DUC_BRANCH do |ok, |
      if ok
        sh 'git', 'checkout', DUC_BRANCH
        sh 'git', 'reset', '--hard', 'upstream/master'
      else
        sh 'git', 'checkout', '-b', DUC_BRANCH, 'upstream/master'
      end
    end
  end

  sh 'tar', '-zcf', DOCSET_ARCHIVE, '--exclude=.DS_Store', DOCSET
  mv DOCSET_ARCHIVE, archive
  mkdir_p versioned_archive.dirname
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
        sh 'git', 'push', '-fu', 'origin', "#{DUC_BRANCH}:#{DUC_BRANCH}"
      end
    end
  end
end

desc 'Send a pull-request'
task :pr => DUC_WORKDIR do
  cd DUC_WORKDIR do
    sh 'git', 'diff', '--exit-code', '--stat', "#{DUC_BRANCH}..upstream/master" do |ok, _res|
      if ok
        puts "Nothing to send a pull-request for."
      else
        sh 'hub', 'pull-request', '-b', "#{DUC_OWNER_UPSTREAM}:master", '-h', "#{DUC_OWNER}:#{DUC_BRANCH}", '-m', `git log -1 --pretty=%s #{DUC_BRANCH}`.chomp
      end
    end
  end
end

desc 'Delete all fetched files and generated files'
task :clean do
  rm_rf [DL_DIR, ICON_FILE, DOCSET, DOCSET_ARCHIVE, FETCH_LOG]
end

task :default => :build
