# Tests Unitarios en NestJS — Página 4
## Módulo 4 · Módulo Users
### Tests completos de UsersService y UsersController

---

## UsersService — bcrypt y manejo de perfil

`UsersService` usa `bcrypt.hash` en `create()` y condicionalmente en
`update()`. Como bcrypt v6 no permite `jest.spyOn`, se mockea el módulo
completo con `jest.mock('bcrypt')` al inicio del archivo.

`updateProfile()` lanza `NotFoundException` directamente (no devuelve
null), por lo que los tests usan `.rejects.toThrow()`.

`update()` no tiene try/catch propio, por lo que no se testa el caso
de error en el repositorio — no ocurriría un retorno null.

---

### `src/users/users.service.spec.ts`

```typescript
jest.mock('bcrypt', () => ({
  hash:    jest.fn(),
  compare: jest.fn(),
}));

import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { NotFoundException } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './user.entity';
import * as bcrypt from 'bcrypt';

const USER_ID      = '33333333-3333-3333-3333-333333333333';
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';

describe('UsersService', () => {
  let service: UsersService;

  const mockUserRepository = {
    create:             jest.fn(),
    save:               jest.fn(),
    findOne:            jest.fn(),
    remove:             jest.fn(),
    createQueryBuilder: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useValue: mockUserRepository },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('create()', () => {

    it('should hash the password before saving', async () => {
      const hashedPwd = '$2b$10$hashedvalue';
      (bcrypt.hash as jest.Mock).mockResolvedValue(hashedPwd);
      mockUserRepository.create.mockReturnValue({ password: hashedPwd });
      mockUserRepository.save.mockResolvedValue({ id: USER_ID, password: hashedPwd });

      const result = await service.create({ username: 'ana', email: 'ana@test.com', password: 'plaintext' });
      expect(result?.password).toBe(hashedPwd);
      expect(result?.password).not.toBe('plaintext');
    });

    it('should call bcrypt.hash with the plain password and salt 10', async () => {
      (bcrypt.hash as jest.Mock).mockResolvedValue('$2b$10$hash');
      mockUserRepository.create.mockReturnValue({});
      mockUserRepository.save.mockResolvedValue({ id: USER_ID });

      await service.create({ username: 'ana', email: 'ana@test.com', password: 'mipassword' });
      expect(bcrypt.hash).toHaveBeenCalledWith('mipassword', 10);
    });

    it('should return the created user', async () => {
      const mockUser = { id: USER_ID, username: 'luis', email: 'luis@test.com', isActive: true };
      (bcrypt.hash as jest.Mock).mockResolvedValue('$2b$10$hash');
      mockUserRepository.create.mockReturnValue(mockUser);
      mockUserRepository.save.mockResolvedValue(mockUser);

      expect(await service.create({ username: 'luis', email: 'luis@test.com', password: 'pass' })).toEqual(mockUser);
    });

    it('should return null when repository throws', async () => {
      (bcrypt.hash as jest.Mock).mockResolvedValue('$2b$10$hash');
      mockUserRepository.create.mockReturnValue({});
      mockUserRepository.save.mockRejectedValue(new Error('Duplicate username'));

      expect(await service.create({ username: 'dup', email: 'dup@test.com', password: 'pass' })).toBeNull();
    });

  });

  describe('findOne()', () => {

    it('should return user when it exists', async () => {
      const mockUser = { id: USER_ID, username: 'ana' };
      mockUserRepository.findOne.mockResolvedValue(mockUser);

      const result = await service.findOne(USER_ID);
      expect(result).toEqual(mockUser);
      expect(mockUserRepository.findOne).toHaveBeenCalledWith({ where: { id: USER_ID } });
    });

    it('should return null when user does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      expect(await service.findOne(NOT_FOUND_ID)).toBeNull();
    });

    it('should return null when repository throws', async () => {
      mockUserRepository.findOne.mockRejectedValue(new Error('DB error'));
      expect(await service.findOne(USER_ID)).toBeNull();
    });

  });

  describe('findByEmail()', () => {

    it('should return user by email', async () => {
      mockUserRepository.findOne.mockResolvedValue({ id: USER_ID, email: 'ana@test.com' });
      const result = await service.findByEmail('ana@test.com');
      expect(mockUserRepository.findOne).toHaveBeenCalledWith({ where: { email: 'ana@test.com' } });
      expect(result).toBeDefined();
    });

    it('should return null when email does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      expect(await service.findByEmail('noexiste@test.com')).toBeNull();
    });

  });

  describe('findByUsername()', () => {

    it('should return user by username', async () => {
      mockUserRepository.findOne.mockResolvedValue({ id: USER_ID, username: 'ana' });
      const result = await service.findByUsername('ana');
      expect(mockUserRepository.findOne).toHaveBeenCalledWith({ where: { username: 'ana' } });
      expect(result).toBeDefined();
    });

    it('should return null when username does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      expect(await service.findByUsername('fantasma')).toBeNull();
    });

  });

  describe('update()', () => {

    it('should return null when user does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      expect(await service.update(NOT_FOUND_ID, { username: 'nuevo' })).toBeNull();
    });

    it('should not hash when password is not provided', async () => {
      mockUserRepository.findOne.mockResolvedValue({ id: USER_ID, username: 'ana' });
      mockUserRepository.save.mockResolvedValue({ id: USER_ID, username: 'ana-nueva' });

      await service.update(USER_ID, { username: 'ana-nueva' });
      expect(bcrypt.hash).not.toHaveBeenCalled();
    });

    it('should hash new password when provided', async () => {
      mockUserRepository.findOne.mockResolvedValue({ id: USER_ID, password: '$2b$10$viejo' });
      (bcrypt.hash as jest.Mock).mockResolvedValue('$2b$10$nuevo_hash');
      mockUserRepository.save.mockResolvedValue({ id: USER_ID, password: '$2b$10$nuevo_hash' });

      const result = await service.update(USER_ID, { password: 'nueva' });
      expect(bcrypt.hash).toHaveBeenCalledWith('nueva', 10);
      expect(result?.password).toBe('$2b$10$nuevo_hash');
    });

    it('should return the updated user', async () => {
      const updatedUser = { id: USER_ID, username: 'ana-v2' };
      mockUserRepository.findOne.mockResolvedValue({ id: USER_ID, username: 'ana' });
      mockUserRepository.save.mockResolvedValue(updatedUser);

      expect(await service.update(USER_ID, { username: 'ana-v2' })).toEqual(updatedUser);
    });

  });

  describe('remove()', () => {

    it('should return null when user does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      expect(await service.remove(NOT_FOUND_ID)).toBeNull();
    });

    it('should call repository.remove with the found user', async () => {
      const mockUser = { id: USER_ID, username: 'ana' };
      mockUserRepository.findOne.mockResolvedValue(mockUser);
      mockUserRepository.remove.mockResolvedValue(mockUser);

      await service.remove(USER_ID);
      expect(mockUserRepository.remove).toHaveBeenCalledWith(mockUser);
    });

    it('should return the removed user', async () => {
      const mockUser = { id: USER_ID, username: 'ana' };
      mockUserRepository.findOne.mockResolvedValue(mockUser);
      mockUserRepository.remove.mockResolvedValue(mockUser);

      expect(await service.remove(USER_ID)).toEqual(mockUser);
    });

  });

  describe('updateProfile()', () => {

    it('should throw NotFoundException when user does not exist', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      await expect(service.updateProfile(NOT_FOUND_ID, 'foto.jpg'))
        .rejects.toThrow(NotFoundException);
    });

    it('should throw NotFoundException with correct message', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);
      await expect(service.updateProfile(USER_ID, 'foto.jpg')).rejects.toThrow('User not found');
    });

    it('should update profile and return user', async () => {
      const mockUser = { id: USER_ID, username: 'ana', profile: null };
      mockUserRepository.findOne.mockResolvedValue(mockUser);
      mockUserRepository.save.mockResolvedValue({ ...mockUser, profile: 'nueva-foto.jpg' });

      const result = await service.updateProfile(USER_ID, 'nueva-foto.jpg');
      expect(result).toHaveProperty('profile', 'nueva-foto.jpg');
    });

    it('should save user with updated profile path', async () => {
      const mockUser = { id: USER_ID, profile: 'vieja.jpg' };
      mockUserRepository.findOne.mockResolvedValue(mockUser);
      mockUserRepository.save.mockResolvedValue({ ...mockUser, profile: 'nueva.jpg' });

      await service.updateProfile(USER_ID, 'nueva.jpg');
      expect(mockUserRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({ profile: 'nueva.jpg' })
      );
    });

  });

});
```

---

## UsersController — guard a nivel de clase y subida de archivos

`UsersController` tiene `@UseGuards(JwtAuthGuard)` en la clase, no en
cada método. El módulo de testing necesita `.overrideGuard()`.

`create()` y `findAll()` no comprueban si el servicio devuelve null
(no tienen lógica de error propia), mientras que `findOne()`, `update()`
y `remove()` sí lanzan `NotFoundException`.

`uploadProfile()` usa `FileInterceptor` con `diskStorage`. En el test
el interceptor no se ejecuta — se pasa el archivo directamente como
parámetro, con un objeto que simula el `Express.Multer.File` completo.

---

### `src/users/users.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, BadRequestException } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';

const USER_ID      = '33333333-3333-3333-3333-333333333333';
const NOT_FOUND_ID = '99999999-9999-9999-9999-999999999999';

describe('UsersController', () => {
  let controller: UsersController;

  const mockUsersService = {
    create:        jest.fn(),
    findAll:       jest.fn(),
    findOne:       jest.fn(),
    update:        jest.fn(),
    remove:        jest.fn(),
    updateProfile: jest.fn(),
  };

  const mockFile: Express.Multer.File = {
    fieldname:    'profile',
    originalname: 'foto.jpg',
    encoding:     '7bit',
    mimetype:     'image/jpeg',
    destination:  './public/profile',
    filename:     '1749600000000-foto.jpg',
    path:         './public/profile/1749600000000-foto.jpg',
    size:         1000,
    buffer:       Buffer.from(''),
    stream:       null as any,
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [{ provide: UsersService, useValue: mockUsersService }],
    })
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: () => true })
      .compile();

    controller = module.get<UsersController>(UsersController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('create()', () => {

    it('should return SuccessResponseDto with created user', async () => {
      const dto      = { username: 'ana', email: 'ana@test.com', password: 'pass' };
      const mockUser = { id: USER_ID, username: 'ana', email: 'ana@test.com' };
      mockUsersService.create.mockResolvedValue(mockUser);

      const result = await controller.create(dto);
      expect(result).toEqual({ success: true, message: 'User created successfully', data: mockUser });
    });

    it('should call usersService.create with the dto', async () => {
      const dto = { username: 'luis', email: 'luis@test.com', password: 'p' };
      mockUsersService.create.mockResolvedValue({ id: USER_ID });
      await controller.create(dto);
      expect(mockUsersService.create).toHaveBeenCalledWith(dto);
    });

  });

  describe('findAll()', () => {

    it('should return SuccessResponseDto with user list', async () => {
      const mockPagination = {
        items: [{ id: USER_ID, username: 'ana' }],
        meta: { currentPage: 1, totalPages: 1, itemCount: 1, totalItems: 1, itemsPerPage: 10 },
      };
      mockUsersService.findAll.mockResolvedValue(mockPagination);

      const result = await controller.findAll({ page: 1, limit: 10 });
      expect(result.success).toBe(true);
      expect(result.data).toEqual(mockPagination);
    });

    it('should call usersService.findAll with the query', async () => {
      mockUsersService.findAll.mockResolvedValue({ items: [], meta: {} });
      const query = { page: 2, limit: 5 };
      await controller.findAll(query);
      expect(mockUsersService.findAll).toHaveBeenCalledWith(query);
    });

  });

  describe('findOne()', () => {

    it('should return SuccessResponseDto with user', async () => {
      const mockUser = { id: USER_ID, username: 'ana' };
      mockUsersService.findOne.mockResolvedValue(mockUser);

      const result = await controller.findOne(USER_ID);
      expect(result).toEqual({ success: true, message: 'User retrieved successfully', data: mockUser });
    });

    it('should throw NotFoundException when user does not exist', async () => {
      mockUsersService.findOne.mockResolvedValue(null);
      await expect(controller.findOne(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

    it('should call usersService.findOne with the correct id', async () => {
      mockUsersService.findOne.mockResolvedValue({ id: USER_ID });
      await controller.findOne(USER_ID);
      expect(mockUsersService.findOne).toHaveBeenCalledWith(USER_ID);
    });

  });

  describe('update()', () => {

    it('should return SuccessResponseDto with updated user', async () => {
      const mockUser = { id: USER_ID, username: 'ana-v2' };
      mockUsersService.update.mockResolvedValue(mockUser);

      const result = await controller.update(USER_ID, { username: 'ana-v2' });
      expect(result).toEqual({ success: true, message: 'User updated successfully', data: mockUser });
    });

    it('should throw NotFoundException when user does not exist', async () => {
      mockUsersService.update.mockResolvedValue(null);
      await expect(controller.update(NOT_FOUND_ID, { username: 'x' }))
        .rejects.toThrow(NotFoundException);
    });

    it('should call usersService.update with correct id and dto', async () => {
      mockUsersService.update.mockResolvedValue({ id: USER_ID });
      const dto = { username: 'nuevo' };
      await controller.update(USER_ID, dto);
      expect(mockUsersService.update).toHaveBeenCalledWith(USER_ID, dto);
    });

  });

  describe('remove()', () => {

    it('should return SuccessResponseDto when user is deleted', async () => {
      const mockUser = { id: USER_ID, username: 'ana' };
      mockUsersService.remove.mockResolvedValue(mockUser);

      const result = await controller.remove(USER_ID);
      expect(result.success).toBe(true);
      expect(result.data).toEqual(mockUser);
    });

    it('should throw NotFoundException when user does not exist', async () => {
      mockUsersService.remove.mockResolvedValue(null);
      await expect(controller.remove(NOT_FOUND_ID)).rejects.toThrow(NotFoundException);
    });

  });

  describe('uploadProfile()', () => {

    it('should return SuccessResponseDto with updated user', async () => {
      const mockUser = { id: USER_ID, profile: mockFile.filename };
      mockUsersService.updateProfile.mockResolvedValue(mockUser);

      const result = await controller.uploadProfile(USER_ID, mockFile);
      expect(result).toEqual({ success: true, message: 'Profile image updated', data: mockUser });
    });

    it('should call usersService.updateProfile with id and filename', async () => {
      mockUsersService.updateProfile.mockResolvedValue({ id: USER_ID });
      await controller.uploadProfile(USER_ID, mockFile);
      expect(mockUsersService.updateProfile).toHaveBeenCalledWith(USER_ID, mockFile.filename);
    });

    it('should throw BadRequestException when no file is provided', async () => {
      await expect(controller.uploadProfile(USER_ID, undefined as any))
        .rejects.toThrow(BadRequestException);
    });

    it('should throw BadRequestException with correct message', async () => {
      await expect(controller.uploadProfile(USER_ID, null as any))
        .rejects.toThrow('Profile image is required');
    });

    it('should not call updateProfile when file is missing', async () => {
      try { await controller.uploadProfile(USER_ID, undefined as any); } catch {}
      expect(mockUsersService.updateProfile).not.toHaveBeenCalled();
    });

  });

});
```
