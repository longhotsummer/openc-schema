require 'rspec/core/rake_task'
RSpec::Core::RakeTask.new(:spec)

task :default => :spec

require 'json'

desc 'Rewrite JSON files with consistent formatting'
task :format_json do
  [
    ['schemas'],
    ['schemas', 'includes'],
    ['spec', '**'],
  ].each do |parts|
    Dir.glob(File.join(*parts, '*.json')).each do |path|
      data = JSON.parse(File.read(path))
      File.open(path, 'w') do |f|
        f.write(JSON.pretty_generate(data))
      end
    end
  end
end

desc 'Write schema files with embedded references'
task :build do
  # @see https://github.com/influencemapping/whos_got_dirt-gem/blob/master/Rakefile

  def define(name, path, definitions)
    unless definitions.key?(name)
      definitions[name] = {} # to avoid recursion
      definitions[name] = process_schema(path, definitions)
    end
  end

  def process_value(value, path, definitions)
    if value.key?('$ref')
      ref = value['$ref']
      unless ref.start_with?('#/definitions/')
        name = File.basename(ref).chomp('.json')
        value['$ref'] = "#/definitions/#{name}"
        define(name, File.expand_path(ref, File.dirname(path)), definitions)
      end
    end
    value
  end

  def process_object(value, path, definitions)
    if value.key?('properties')
      process_properties(value['properties'], path, definitions)
    else
      keyword = (value.keys & ['allOf', 'anyOf', 'oneOf']).first
      if keyword
        value[keyword].each do |subschema|
          process_object(subschema, path, definitions)
        end
      else
        process_value(value, path, definitions)
      end
    end
    value
  end

  def process_properties(properties, path, definitions)
    properties.each do |_,value|
      if value.key?('items')
        process_object(value['items'], path, definitions)
      else
        process_object(value, path, definitions)
      end
    end
    properties
  end

  def process_schema(path, definitions)
    schema = JSON.load(File.read(path))
    if schema.key?('definitions')
      schema['definitions'].each do |_,definition|
        process_object(definition, path, definitions)
      end
      definitions.merge!(schema['definitions'])
    end
    process_object(schema, path, definitions)
  end

  all_definitions = {}

  Dir[File.join('schemas', '*.json')].each do |path|
    definitions = {} # passed by reference
    schema = process_schema(path, definitions).merge('definitions' => definitions)
    all_definitions.merge!(definitions) # cache definitions across schema
    File.open(File.join('build', File.basename(path)), 'w') do |f|
      f.write(JSON.pretty_generate(schema))
    end
  end
end

desc 'Converts a data model in CSV to JSON Schema'
task :csv_to_json_schema do
  require 'csv'
  require 'open-uri'
  require 'set'

  REQUIRED_HEADERS = [
    'Class',
    'Property',
    'Description',
    'Type',
    'Format',
    'Required?',
    'Example',
  ].freeze

  # @see http://json-schema.org/latest/json-schema-core.html#anchor8
  JSON_SCHEMA_TYPES = [
    'array',
    'boolean',
    'integer',
    'null',
    'number',
    'object',
    'string',
  ].freeze

  JSON_SCHEMA_TYPES_RE = Regexp.new(JSON_SCHEMA_TYPES.join('|'))

  DEFINED_CLASSES = Set.new

  REFERENCED_CLASSES = Set.new

  def class_name_to_title(class_name)
    class_name.gsub(/(?<=[a-z])(?=[A-Z])/, ' ')
  end

  def class_name_to_definition_key(class_name)
    class_name.gsub(/(?<=[a-z])(?=[A-Z])/, '_').downcase
  end

  def references_definition?(property)
    property && property['$ref'] && property['$ref'][%r{#/definitions/(.+)}, 1]
  end

  def parse_type(type)
    case type
    when *JSON_SCHEMA_TYPES
      {'type' => type}
    when /\Aarray<(.+)>\z/
      {'type' => 'array', 'items' => parse_type($1)}
    when /\A[A-Z][A-Za-z]+\z/
      definition_key = class_name_to_definition_key(type)
      if DEFINED_CLASSES.include?(type)
        REFERENCED_CLASSES << type
        {'$ref' => "#/definitions/#{definition_key}"}
      elsif File.exist?(File.join('schemas', 'includes', "#{definition_key}.json"))
        {'$ref' => "includes/#{definition_key}.json"}
      else
        raise "unrecognized schema: #{type}"
      end
    else
      types = type.scan(JSON_SCHEMA_TYPES_RE)
      if types.empty?
        raise "unrecognized type: #{type}"
      else
        {'type' => types}
      end      
    end
  end

  url = ENV['url']
  unless url
    abort 'usage: url=<url> rake csv_to_json_schema'
  end

  rows = CSV.parse(open(url).read, headers: true).select{|row| row['Class']}

  missing_headers = REQUIRED_HEADERS - rows[0].headers
  unless missing_headers.empty?
    abort "CSV is missing #{missing_headers.join(',')} headers"
  end

  definition_requireds = {}
  definition_descriptions = {}
  rows.each do |row|
    klass = row['Class']
    DEFINED_CLASSES << klass

    required = row['Required?']
    definition_key = class_name_to_definition_key(klass)

    definition_requireds[definition_key] ||= []
    case required
    when 'Y'
      definition_requireds[definition_key] << row['Property']
    when nil
      # do nothing
    else
      abort "unrecognized required flag: #{required}"
    end

    unless row['Property']
      definition_descriptions[definition_key] = row['Description']
    end
  end

  rows.select!{|row| row['Property']}

  DEFINITIONS = {}
  EXAMPLES = {}
  rows.each do |row|
    property = {}
    if row['Description']
      property['description'] = row['Description']
    end
    property.merge!(parse_type(row['Type']))

    format = row['Format']
    case format
    when 'N/A'
      # do nothing
    when 'date', 'date-time', 'email', 'hostname', 'ipv4', 'ipv6', 'uri'
      property['format'] = format
    when %r{\A/(.+)/\z}
      property['pattern'] = $1
    when %r{, }
      property['enum'] = format.split(', ')
    else # e.g. IANA media type
      $stderr.puts "unrecognized format: #{format}"
    end

    klass = row['Class']
    definition_key = class_name_to_definition_key(klass)

    unless DEFINITIONS.key?(definition_key)
      definition = {
        'title' => class_name_to_title(klass),
      }
      if definition_descriptions.key?(definition_key)
        definition['description'] = definition_descriptions[definition_key]
      end
      DEFINITIONS[definition_key] = definition.merge({
        'type' => 'object',
        'properties' => {},
        'additionalProperties' => false,
      })
      EXAMPLES[definition_key] = {}
    end
    DEFINITIONS[definition_key]['properties'][row['Property']] = property

    example = JSON.load(row['Example'])
    if !example.nil? || references_definition?(property) || references_definition?(property['items'])
      EXAMPLES[definition_key][row['Property']] = example
    end
  end

  DEFINITIONS.each do |key,definition|
    unless definition_requireds[key].empty?
      DEFINITIONS[key]['required'] = definition_requireds[key]
    end
  end

  primary_classes = DEFINED_CLASSES - REFERENCED_CLASSES
  if primary_classes.one?
    primary_definition_key = class_name_to_definition_key(primary_classes.to_a[0])
    primary_definition = DEFINITIONS.delete(primary_definition_key)
    primary_example = EXAMPLES.delete(primary_definition_key)
  else
    abort "couldn't determine primary class (one of #{primary_classes.to_a.join(', ')})"
  end

  schema = {
    '$schema' => 'http://json-schema.org/draft-04/schema#',
  }.merge(primary_definition).merge({
    'definitions' => DEFINITIONS,
  })

  def build_example(example, schema)
    example.each_key do |key|
      property = schema['properties'][key]
      definition_key = references_definition?(property)
      if definition_key
        object = build_example(EXAMPLES[definition_key], DEFINITIONS[definition_key])
        if object.values.any?
          example[key] = object
        else
          example.delete(key)
        end
      else
        definition_key = references_definition?(property['items'])
        if definition_key
          example[key] = [build_example(EXAMPLES[definition_key], DEFINITIONS[definition_key])]
        end
      end
    end
    example
  end

  if ENV['example']
    puts JSON.pretty_generate(build_example(primary_example, schema))
  else
    puts JSON.pretty_generate(schema)
  end
end
