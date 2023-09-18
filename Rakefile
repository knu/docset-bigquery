# frozen_string_literal: true
require 'bundler/setup'
Bundler.require

require 'json'
require 'pathname'
require 'set'
require 'tempfile'
require 'time'
require 'uri'
require 'zlib'
require 'rubygems/version'

def jenkins?
  /jenkins-/.match?(ENV['BUILD_TAG'])
end

def paginate_command(cmd, diff: false)
  case cmd
  when Array
    cmd = cmd.shelljoin
  end

  if $stdout.tty? || (diff && jenkins?)
    pager = (ENV['DIFF_PAGER'] if diff) || ENV['PAGER'] || 'more'
    "#{cmd} | #{pager}"
  else
    cmd
  end
end

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
DOCS_DIR = Pathname(DOCS_URI.host)
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

def current_version
  ENV['BUILD_VERSION'] || extract_version()
end

def previous_version
  ENV['PREVIOUS_VERSION'] || Gem::Version.new(current_version).then { |current_version|
    Pathname.glob("versions/*/#{DOCSET}").map { |path|
      Gem::Version.new(path.parent.basename.to_s)
    }.select { |version|
      version < current_version
    }.max&.to_s
  }
end

def previous_docset
  version = previous_version or raise 'No previous version found'

  "versions/#{version}/#{DOCSET}"
end

def built_docset
  if version = ENV['BUILD_VERSION']
    "versions/#{version}/#{DOCSET}"
  else
    DOCSET
  end
end

def dump_index(docset, out)
  index = File.join(docset, INDEX_RELPATH)

  SQLite3::Database.new(index) do |db|
    db.execute("SELECT name, type, path FROM searchIndex ORDER BY name, type, path") do |row|
      out.puts row.join("\t")
    end
  end

  out.flush
end

desc "Fetch the #{DOCSET_NAME} document files."
task :fetch => %i[fetch:icon fetch:docs]

namespace :fetch do
  task :docs do
    puts 'Downloading %s' % DOCS_URI
    wget_options = %W[
      -nv --append-output #{FETCH_LOG} -N -p -E
      #{'--reject-regex=\?hl=|\?_gl=|://cloud\.google\.com/images/(artwork|backgrounds|home|icons|logos)/|\.md$'}
    ]
    sh 'wget', *wget_options, DOCS_URI.to_s
    sh 'wget', *wget_options, '-r', '--no-parent', (DOCS_URI + 'query-syntax').to_s

    # Google responds with gzip'd asset files despite wget's sending
    # `Accept-Encoding: identity`, and wget has no capability in
    # decoding gzip contents.
    Dir.glob("#{DOCS_DIR}/**/*") { |path|
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

file DOCS_DIR do
  Rake::Task[:'fetch:docs'].invoke
end

file ICON_FILE do
  Rake::Task[:'fetch:icon'].invoke
end

desc 'Build a docset in the current directory.'
task :build => [DOCS_DIR, ICON_FILE] do |t|
  rm_rf [DOCSET, DOCSET_ARCHIVE]

  mkdir_p DOCS_ROOT

  cp 'Info.plist', File.join(DOCSET, 'Contents')
  cp ICON_FILE, DOCSET

  cp_r DOCS_DIR.to_s + '/.', DOCS_ROOT

  version = extract_version or raise "Version unknown"

  puts "Generating docset for #{DOCSET_NAME} #{version}"

  # Index
  db = SQLite3::Database.new(DOCS_INDEX)

  db.execute(<<-SQL)
    CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);
  SQL
  db.execute(<<-SQL)
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

  bad_hrefs = Set[]

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
          begin
            href = e[attr]

            case abs = uri + href
            when URI::HTTP
              # ok
            else
              next
            end
          rescue URI::Error => e
            warn "#{e.message} in #{path}" if bad_hrefs.add?(href)
            next
          end

          abs.path = abs.path.chomp('.md')
          rel = HOST_URI.route_to(abs)
          if rel.host || %r{\A\.\./} === rel.path
            # Rewrite all URLs to those relative from the base URL
            e[attr] = abs
            next
          end
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

      doc.css('.devsite-breadcrumb-list').each(&:remove)
      doc.css('devsite-feedback, devsite-hats-survey, devsite-thumb-rating').each(&:remove)

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
          title = h.xpath('normalize-space(.)')
          index_item.(path, h, 'Section', title)
        }
        if h = doc.at('h3#sql')
          h.xpath('./following-sibling::p[1]/descendant-or-self::code[starts-with(., "#")]').each { |td|
            directive = td.xpath('normalize-space(.)')
            index_item.(path, td, 'Directive', directive)
          }
        end
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
            td.xpath('./code').each { |code|
              case type = code.xpath('normalize-space(.)')
              when /\A([A-Z][A-Z0-9]*)\z/
                index_item.(path, td, 'Type', $1)
              else
                raise "#{path}: Unknown type: #{text}"
              end
            }
          }
        }
      when 'functions-and-operators.html'
        # This page is now a one-page function reference that wraps up
        # the following pages.
      when /(?:utility-|_)functions\.html\z/, 'conversion_rules.html', 'conditional_expressions.html', 'operators.html', 'table-functions-built-in.html'
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
          when /\A(LIKE|IS DISTINCT FROM) operator\z/
            syntax = h.at_xpath('./following-sibling::pre[1]/code').text
            op = syntax[/\b[A-Z]+( \[?[A-Z]+\]?)+/]
            index_item.(path, h, 'Operator', op)
          when /\A(?<func>(?<WORD>[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)(?: \g<WORD>)*)( (?:and|or) \g<func>)*(?: (?<thing>operators?|expr))?\z/
            # 'or' is for 'JSON_EXTRACT or JSON_EXTRACT_SCALAR'
            type =
              case $~[:thing]
              when /operator/
                'Operator'
              else
                h.name == 'h6' ? 'Query' : 'Function'
              end
            title.scan(/(?<func>(?<WORD>[A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)(?: \g<WORD>)*)/) { |name,|
              index_item.(path, h, type, name)
            }
            next
          when 'Logical operators'
            h.xpath('(./following-sibling::p)[1]/code').each { |code|
              case text = code.text
              when 'TRUE', 'FALSE', 'NULL', 'BOOL'
                # skip
              when 'AND', 'OR', 'NOT'
                index_item.(path, h, 'Operator', text)
              else
                raise "#{path}: Unknown logical operator: #{text}"
              end
            }
          when /\A(Arithmetic|Bitwise|Logical|Comparison) operators\z/
            h.xpath('((./following-sibling::*/descendant-or-self::table)[1]/tbody/tr/td[2])/code').each { |code|
              syntax = code.xpath('normalize-space(.)')
              next unless /\bX\b/ === syntax
              op, rop, = syntax.gsub(/\b[XYZ]\b/, '').split
              case op
              when '[NOT]'
                index_item.(path, h, 'Operator', rop)
                index_item.(path, h, 'Operator', "NOT #{rop}")
              else
                if syntax.start_with?(op)
                  index_item.(path, h, 'Operator', "#{op} (Unary)")
                else
                  index_item.(path, h, 'Operator', op)
                end
              end
            }
          when 'Field access operator'
            index_item.(path, h, 'Operator', '.')
          when 'Array subscript operator', 'Struct subscript operator'
            index_item.(path, h, 'Operator', "[] (#{title})")
            h.xpath('./following-sibling::*//li[starts-with(normalize-space(.), "position_keyword(index):")]/ul/li').each { |li|
              case li.at_xpath('./code')&.text
              when /\A([A-Z][A-Z0-9]*(?:[_.][A-Z0-9]+)*)\(/
                index_item.(path, h, 'Function', $1)
              end
            }
          when 'JSON subscript operator'
            index_item.(path, h, 'Operator', "[] (#{title})")
          when 'Date arithmetics operators', 'Interval arithmetic operators'
            # Nothing to link
          when 'Quantified LIKE operator'
            h.xpath('./following-sibling::*//li[starts-with(normalize-space(.), "quantifier:")]/ul/li').each { |li|
              case li.at_xpath('.//code')&.text
              when /\A([A-Z]+)/
                index_item.(path, h, 'Operator', "LIKE #{$1}")
              end
            }
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
      when 'procedural-language.html'
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
          when /\AArray subqueries\b/
            index_item.(path, h, 'Query', 'ARRAY')
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
          when 'Syntax'
            next
          when /\A\w+_option_list\z/
            h.xpath('./following-sibling::*').each { |e|
              case e.name
              when 'h1'..h.name
                break
              when 'table'
                e.xpath("self::table[.//tr/th[1][normalize-space(.)='Options' or normalize-space(.)='NAME']]//tr/td[1][./code]").each { |td|
                  name = td.xpath('normalize-space(./code)')
                  warn "garbage found in option #{name}" if name.sub!(/\A\S+\K\s.+/s, '')
                  index_item.(path, td, 'Option', name)
                }
              end
            }
          when 'Defining the window frame clause'
            code, = h.xpath('./following-sibling::pre[1]/code')
            code.text.scan(/[A-Z]+(?: [A-Z]+)*/) { |query|
              index_item.(path, code, 'Query', query) unless query == 'AND'
            }
          when /\A(?<ws>(?<w>[A-Z]+)(?: \g<w>)*) (?<t>statement|keyword|[cC]lause|operator)(?: and \g<ws> \k<t>)?\z/
            type =
              case $~[:t]
              when 'statement'
                'Statement'
              when 'operator'
                'Operator'
              else
                'Query'
              end
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
              when /\A(?:(?:SELECT|CREATE|DROP|ASSERT)\b|(?:)\z)/
                'Statement'
              when /\bJOIN\z/, 'UNION', 'INTERSECT', 'EXCEPT', 'FOR SYSTEM_TIME AS OF',
                  'ROWS', 'RANGE', 'OPTIONS'
                'Query'
              else
                case File.basename(path)
                when /\Abigqueryml-/
                  'Option'
                else
                  raise "#{path}: Unknown directive: #{title}"
                end
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
                    'CREATE EXTERNAL TABLE', 'EXPORT DATA', 'DROP TABLE FUNCTION',
                    'GRANT', 'REVOKE'],
    'Query' => ['JOIN', 'INNER JOIN',
                'UNION', 'INTERSECT', 'EXCEPT',
                'FOR SYSTEM_TIME AS OF',
                'GROUP BY', 'LIMIT', 'OFFSET',
                'ARRAY', 'IN', 'EXISTS',
                'ROWS', 'RANGE', 'BETWEEN',
                'UNBOUNDED FOLLOWING', 'FOLLOWING',
                'CURRENT ROW',
                'OPTIONS'],
    'Function' => ['CAST', 'SAFE_CAST',
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
    'Type' => ['INT64', 'BIGINT', 'FLOAT64', 'NUMERIC', 'BOOL', 'STRING', 'BYTES',
               'DATE', 'DATETIME', 'TIME', 'TIMESTAMP',
               'ARRAY', 'STRUCT', 'BIGNUMERIC'],
    'Operator' => ['+', '~ (Unary)', '^', '<=', '!=', '<>', '.', '[] (Array subscript operator)', '||',
                   'BETWEEN', 'NOT LIKE', 'LIKE SOME', 'AND', 'OR', 'NOT',
                   'IN', 'IS', 'IS [NOT] DISTINCT FROM', 'IS [NOT] LIKE', 'UNNEST'],
    'Option' => ['max_batching_rows', 'overwrite', 'field_delimiter', 'friendly_name',
                 'HPARAM_TUNING_ALGORITHM', 'MODEL_TYPE', 'TIME_SERIES_TIMESTAMP_COL',
                 'OPTIMIZER', 'TRANSFORM'],
    'Section' => ['GCM', 'Loops', 'SQL UDFs', 'JSON subscript operator']
  }.each { |type, names|
    names.each { |name|
      assert_exists.(name: name, type: type)
    }
  }

  db.close

  sh 'tar', '-zcf', DOCSET_ARCHIVE, '--exclude=.DS_Store', DOCSET

  mkdir_p "versions/#{version}/#{DOCSET}"
  sh 'rsync', '-a', '--exclude=.DS_Store', '--delete', "#{DOCSET}/", "versions/#{version}/#{DOCSET}/"

  puts "Finished creating #{DOCSET} #{version}"

  system paginate_command('rake diff:index', diff: true)
end

task :dump do
  system paginate_command('rake dump:index')
end

namespace :dump do
  desc 'Dump the index.'
  task :index do
    dump_index(built_docset, $stdout)
  end
end

task :diff do
  system paginate_command('rake diff:index diff:docs', diff: true)
end

namespace :diff do
  desc 'Show the differences in the index from an installed version.'
  task :index do
    Tempfile.create(['old', '.txt']) do |otxt|
      Tempfile.create(['new', '.txt']) do |ntxt|
        dump_index(previous_docset, otxt)
        otxt.close
        dump_index(built_docset, ntxt)
        ntxt.close

        puts "Diff in document indexes:"
        sh 'diff', '-U3', otxt.path, ntxt.path do
          # ignore status
        end
      end
    end
  end

  desc 'Show the differences in the docs from an installed version.'
  task :docs do
    old_root = File.join(previous_docset, ROOT_RELPATH)

    puts "Diff in document files:"
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

  cp DOCSET_ARCHIVE, archive
  mkdir_p versioned_archive.dirname
  cp archive, versioned_archive

  specific_versions = nil

  puts "Updating #{docset_json}"
  File.open(docset_json, 'r+') { |f|
    json = JSON.parse(f.read)
    json['version'] = version
    specific_version = {
      'version' => version,
      'archive' => versioned_archive.relative_path_from(workdir).to_s
    }
    specific_versions = json['specific_versions'] = [specific_version] | json['specific_versions']
    f.rewind
    f.puts JSON.pretty_generate(json, indent: "    ")
    f.truncate(f.tell)
  }

  cd workdir.to_s do
    json_path = docset_json.relative_path_from(workdir).to_s

    if system(*%W[git diff --exit-code --quiet #{json_path}])
      puts "Nothing to commit."
      next
    end

    sh paginate_command(%W[git diff #{json_path}], diff: true)

    sh 'git', 'add', *[archive, versioned_archive, docset_json].map { |path|
      path.relative_path_from(workdir).to_s
    }
    sh 'git', 'commit', '-m', "Update #{DOCSET_NAME} docset to #{version}"
    sh 'git', 'push', '-fu', 'origin', "#{DUC_BRANCH}:#{DUC_BRANCH}"

    last_version = specific_versions.dig(1, 'version')
    puts "Diff to the latest version #{last_version}:"
    sh({ 'PREVIOUS_VERSION' => last_version }, paginate_command("rake diff:index", diff: true))

    puts "New docset is committed and pushed to #{DUC_OWNER}:#{DUC_BRANCH}.  To send a PR, go to the following URL:"
    puts "\t" + "#{DUC_REPO_UPSTREAM.delete_suffix(".git")}/compare/master...#{DUC_OWNER}:#{DUC_BRANCH}?expand=1"
  end
end

desc 'Send a pull-request'
task :pr => DUC_WORKDIR do
  cd DUC_WORKDIR do
    sh(*%W[git diff --exit-code --stat #{DUC_BRANCH}..upstream/master]) do |ok, _res|
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
  rm_rf [DOCS_DIR, ICON_FILE, DOCSET, DOCSET_ARCHIVE, FETCH_LOG]
end

task :default => :build
