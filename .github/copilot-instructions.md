# GitHub Copilot Instructions for Proxmox Module

## Repository Overview

This repository contains a **Proxmox module for FOSSBilling**, a comprehensive billing system integration that enables automated provisioning and management of Proxmox VE virtual machines and containers. The module is currently in **0.1.0 preview** and under active development.

### Core Purpose
- Integrate Proxmox VE hypervisor with FOSSBilling billing system
- Automate VM/LXC provisioning based on customer orders
- Provide client privilege separation and multi-tenant management
- Enable IP address management (IPAM) and network configuration
- Support multiple Proxmox servers with load balancing

## Architecture & Code Structure

### Main Components

#### Core Service Classes
- `src/Service.php` - Main service class implementing FOSSBilling service interface
- `src/Api/Admin.php` - Administrative API endpoints for server management
- `src/Api/Client.php` - Client-facing API for VM management

#### Trait-Based Architecture
The module uses PHP traits to organize functionality:
- `src/ProxmoxVM.php` - VM lifecycle management (start, stop, create, delete)
- `src/ProxmoxIPAM.php` - IP address and network management
- `src/ProxmoxServer.php` - Proxmox server communication and management
- `src/ProxmoxTemplates.php` - VM and LXC template handling
- `src/ProxmoxAuthentication.php` - Authentication and API client management

#### Web Interface
- `src/html_admin/` - Admin panel templates and assets
- `src/html_client/` - Client portal interfaces
- `src/assets/` - CSS, JavaScript, and other static assets

### Database Conventions

#### Table Naming
- All module tables prefixed with `service_proxmox_`
- Example tables: `service_proxmox_server`, `service_proxmox_vm_config_template`

#### Common Patterns
- Use RedBeanPHP ORM for database operations
- Store JSON configuration in `config` fields
- Always include `created_at` and `updated_at` timestamps
- Use `active` boolean field for soft deletion

## Development Environment

### Prerequisites
- PHP 8.0+ (supports 8.1 and 8.2)
- FOSSBilling 0.5.5+ installed
- Composer for dependency management
- Access to Proxmox VE server for testing

### Dependencies
- `pve2-api-client/pve2-api-client` - Proxmox API communication
- `symfony/finder` - File system operations
- PHPUnit for testing
- PHPStan for static analysis

### Setup Commands
```bash
composer install
vendor/bin/phpunit tests/
vendor/bin/phpstan analyse
```

## Code Patterns & Conventions

### API Endpoint Structure
```php
public function endpoint_name($data)
{
    // 1. Validate required parameters
    $required = array('param1' => 'Error message');
    $this->di['validator']->checkRequiredParamsForArray($required, $data);
    
    // 2. Load related models
    $model = $this->di['db']->findOne('table_name', 'condition');
    
    // 3. Perform business logic
    $result = $this->getService()->perform_action($data);
    
    // 4. Return result
    return $result;
}
```

### Proxmox API Integration
```php
// Get Proxmox instance
$server = $this->di['db']->findOne('service_proxmox_server', 'id=:id', [':id' => $server_id]);
$proxmox = $this->getProxmoxInstance($server);

// Authenticate and make API calls
if ($proxmox->login()) {
    $result = $proxmox->get('/api2/json/endpoint');
    // or
    $result = $proxmox->post('/api2/json/endpoint', $data);
}
```

### Error Handling
- Always wrap Proxmox API calls in try-catch blocks
- Use `\Box_Exception` for user-facing errors
- Log errors using `$this->di['logger']->error()`
- Validate server connectivity before operations

### Configuration Management
```php
// Product configuration (JSON stored in database)
$product = $this->di['db']->load('product', $order->product_id);
$config = json_decode($product->config, true);

// Access configuration values
$vm_cores = $config['cpu_cores'];
$vm_memory = $config['vmmemory'];
```

## Testing Practices

### Test Structure
- Tests located in `tests/Serviceproxmox/`
- Use PHPUnit with FOSSBilling's `\BBTestCase` base class
- Mock external dependencies (database, Proxmox API)

### Test Patterns
```php
public function testMethodName()
{
    // Setup mocks
    $dbMock = $this->getMockBuilder('\\Box_Database')->getMock();
    
    // Configure expectations
    $dbMock->expects($this->once())
           ->method('findOne')
           ->willReturn($mockModel);
    
    // Test execution
    $result = $this->service->methodName($input);
    
    // Assertions
    $this->assertEquals($expected, $result);
}
```

## Security Considerations

### Authentication
- Support both username/password and API token authentication
- Store passwords securely, clear them from responses
- Validate server certificates in production

### Input Validation
- Always validate user input using `$this->di['validator']`
- Sanitize data before database storage
- Escape data for shell commands

### Privilege Separation
- Clients can only access their own VMs
- Admin API endpoints require admin privileges
- Use FOSSBilling's built-in authorization

## Common Development Tasks

### Adding New API Endpoints

1. **Admin Endpoint**: Add to `src/Api/Admin.php`
2. **Client Endpoint**: Add to `src/Api/Client.php`
3. **Service Method**: Implement logic in `src/Service.php` or appropriate trait
4. **Tests**: Create corresponding test methods

### Database Schema Changes
1. Create migration in `src/migrations/`
2. Update relevant model handling
3. Test migration with existing data

### Adding Proxmox Features
1. Study Proxmox API documentation
2. Implement in appropriate trait (VM, IPAM, etc.)
3. Add corresponding admin/client interfaces
4. Include comprehensive error handling

### UI Development
1. Admin interfaces go in `src/html_admin/`
2. Client interfaces go in `src/html_client/`
3. Use FOSSBilling's templating system
4. Include proper CSRF protection

## Module Lifecycle

### Installation Process
1. User uploads module to FOSSBilling
2. FOSSBilling calls Service->install()
3. Database tables created via migrations
4. Module configuration set up

### Order Processing
1. Customer places order for Proxmox product
2. FOSSBilling calls Service->create()
3. Module finds suitable server, allocates resources
4. VM/LXC created via Proxmox API
5. Service model stored with VM details

### Ongoing Management
- Start/stop/reboot operations via client interface
- Admin monitoring and server management
- Automatic backups and resource monitoring

## Debugging Tips

### Common Issues
- **Proxmox API errors**: Check server connectivity and credentials
- **VM creation failures**: Verify template availability and resource limits
- **Database errors**: Check table structure and migrations

### Logging
```php
// Use FOSSBilling's logger
$this->di['logger']->info('Operation successful: %s', $details);
$this->di['logger']->error('Operation failed: %s', $error_message);
```

### Development Mode
- Enable detailed error reporting in FOSSBilling
- Use PHPStan for static analysis
- Test with non-production Proxmox servers

## Contributing Guidelines

### Code Style
- Follow PSR-12 coding standards
- Use meaningful variable and method names
- Include PHPDoc comments for public methods
- Keep methods focused and small

### Pull Request Process
1. Create feature branch from main
2. Implement changes with tests
3. Run PHPStan and PHPUnit
4. Update documentation if needed
5. Submit PR with clear description

### Known TODOs
- Better error handling for unexpected Proxmox responses
- Improved VM allocation procedures
- CloudInit provisioning support
- Enhanced template management
- Better client usability features

## API Usage Examples

### Creating a VM
```php
// This happens automatically when customer places order
$order = /* customer order */;
$service = $this->create($order);
// Results in VM creation on allocated Proxmox server
```

### Managing VM State
```php
// Start VM
$this->vm_start($order, $service);

// Stop VM
$this->vm_shutdown($order, $service);

// Get VM status
$status = $this->vm_status($order, $service);
```

### IPAM Operations
```php
// Get available IP ranges
$ranges = $this->get_ip_ranges();

// Allocate IP to client
$ip = $this->allocate_ip($client_id, $vlan_id);
```

Remember: This module is in preview state, so expect frequent changes and improvements. Always test thoroughly in development environments before deploying changes.