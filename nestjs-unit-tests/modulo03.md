# Tests Unitarios en NestJS — Página 3
## Módulo 3 · Módulo Auth
### Tests completos de AuthService y AuthController

---

## AuthService — dependencias y estrategia de mock

`AuthService` tiene tres dependencias: `UsersService`, `JwtService`
y `bcrypt`. Las dos primeras se mockean con objetos planos en el
módulo de testing. `bcrypt` requiere `jest.mock()` a nivel de archivo
porque en bcrypt v6 sus propiedades no son reconfigurables con
`jest.spyOn`.

`login()` busca el usuario, compara contraseña con `bcrypt.compare` y
firma el token con `JwtService.sign`.
`register()` delega la creación al `UsersService.create` y firma el
token si se creó correctamente.

La constante `USER_ID` representa un UUID válido como los que genera
TypeORM en producción.

---

### `src/auth/auth.service.spec.ts`

```typescript
jest.mock('bcrypt', () => ({
  compare: jest.fn(),
  hash:    jest.fn(),
}));

import { Test, TestingModule } from '@nestjs/testing';
import { AuthService } from './auth.service';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';

const USER_ID = '33333333-3333-3333-3333-333333333333';

describe('AuthService', () => {
  let service: AuthService;

  const mockUsersService = {
    findByUsername: jest.fn(),
    create:         jest.fn(),
  };

  const mockJwtService = {
    sign: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: UsersService, useValue: mockUsersService },
        { provide: JwtService,   useValue: mockJwtService },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('login()', () => {

    it('should return null when user does not exist', async () => {
      mockUsersService.findByUsername.mockResolvedValue(null);
      expect(await service.login({ username: 'noexiste', password: '1234' })).toBeNull();
    });

    it('should return null when password is incorrect', async () => {
      const mockUser = { id: USER_ID, username: 'admin', password: '$2b$10$hash' };
      mockUsersService.findByUsername.mockResolvedValue(mockUser);
      (bcrypt.compare as jest.Mock).mockResolvedValue(false);
      expect(await service.login({ username: 'admin', password: 'wrong' })).toBeNull();
    });

    it('should return a JWT token on successful login', async () => {
      const mockUser = { id: USER_ID, username: 'admin', password: '$2b$10$hash' };
      mockUsersService.findByUsername.mockResolvedValue(mockUser);
      (bcrypt.compare as jest.Mock).mockResolvedValue(true);
      mockJwtService.sign.mockReturnValue('jwt.token.aqui');

      const result = await service.login({ username: 'admin', password: 'correcta' });
      expect(result).toBe('jwt.token.aqui');
    });

    it('should call jwtService.sign with correct payload', async () => {
      const mockUser = { id: USER_ID, username: 'maria', password: '$2b$10$hash' };
      mockUsersService.findByUsername.mockResolvedValue(mockUser);
      (bcrypt.compare as jest.Mock).mockResolvedValue(true);
      mockJwtService.sign.mockReturnValue('token');

      await service.login({ username: 'maria', password: 'pass' });
      expect(mockJwtService.sign).toHaveBeenCalledWith({ id: USER_ID, username: 'maria' });
    });

    it('should return null on unexpected error', async () => {
      mockUsersService.findByUsername.mockRejectedValue(new Error('DB connection error'));
      expect(await service.login({ username: 'admin', password: 'pass' })).toBeNull();
    });

  });

  describe('register()', () => {

    it('should return null when user creation fails', async () => {
      mockUsersService.create.mockResolvedValue(null);
      expect(await service.register({ username: 'nuevo', password: 'pass123', email: 'nuevo@test.com' })).toBeNull();
    });

    it('should return a JWT token on successful registration', async () => {
      const mockUser = { id: USER_ID, username: 'nuevo', email: 'nuevo@test.com' };
      mockUsersService.create.mockResolvedValue(mockUser);
      mockJwtService.sign.mockReturnValue('registro.token');

      const result = await service.register({ username: 'nuevo', password: 'pass123', email: 'nuevo@test.com' });
      expect(result).toBe('registro.token');
    });

    it('should call usersService.create with the dto', async () => {
      const dto = { username: 'nuevo', password: 'pass123', email: 'nuevo@test.com' };
      mockUsersService.create.mockResolvedValue(null);
      await service.register(dto);
      expect(mockUsersService.create).toHaveBeenCalledWith(dto);
    });

    it('should not call jwtService.sign when user creation fails', async () => {
      mockUsersService.create.mockResolvedValue(null);
      await service.register({ username: 'x', password: 'y', email: 'z@z.com' });
      expect(mockJwtService.sign).not.toHaveBeenCalled();
    });

  });

});
```

---

## AuthController — excepciones según resultado del servicio

`AuthController` depende solo de `AuthService`. No tiene guards a nivel
de clase. `login()` lanza `UnauthorizedException` si el servicio
devuelve `null`; `register()` lanza `BadRequestException`. Los tests
verifican tanto el tipo de excepción como el mensaje exacto.

---

### `src/auth/auth.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UnauthorizedException, BadRequestException } from '@nestjs/common';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';

describe('AuthController', () => {
  let controller: AuthController;

  const mockAuthService = {
    login:    jest.fn(),
    register: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [AuthController],
      providers: [
        { provide: AuthService, useValue: mockAuthService },
      ],
    }).compile();

    controller = module.get<AuthController>(AuthController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('login()', () => {

    it('should return access_token on successful login', async () => {
      mockAuthService.login.mockResolvedValue('mi.jwt.token');

      const result = await controller.login({ username: 'admin', password: 'pass' });
      expect(result).toEqual({
        success: true,
        message: 'Login successful',
        data:    { access_token: 'mi.jwt.token' },
      });
    });

    it('should throw UnauthorizedException when credentials are invalid', async () => {
      mockAuthService.login.mockResolvedValue(null);
      await expect(controller.login({ username: 'x', password: 'wrong' }))
        .rejects.toThrow(UnauthorizedException);
    });

    it('should throw UnauthorizedException with correct message', async () => {
      mockAuthService.login.mockResolvedValue(null);
      await expect(controller.login({ username: 'x', password: 'y' }))
        .rejects.toThrow('Invalid credentials');
    });

  });

  describe('register()', () => {

    it('should return access_token on successful registration', async () => {
      mockAuthService.register.mockResolvedValue('nuevo.jwt.token');

      const result = await controller.register({ username: 'nuevo', password: 'pass', email: 'n@n.com' });
      expect(result).toEqual({
        success: true,
        message: 'Registration successful',
        data:    { access_token: 'nuevo.jwt.token' },
      });
    });

    it('should throw BadRequestException when registration fails', async () => {
      mockAuthService.register.mockResolvedValue(null);
      await expect(controller.register({ username: 'x', password: 'y', email: 'z@z.com' }))
        .rejects.toThrow(BadRequestException);
    });

    it('should throw BadRequestException with correct message', async () => {
      mockAuthService.register.mockResolvedValue(null);
      await expect(controller.register({ username: 'x', password: 'y', email: 'z@z.com' }))
        .rejects.toThrow('Failed to register user');
    });

  });

});
```
